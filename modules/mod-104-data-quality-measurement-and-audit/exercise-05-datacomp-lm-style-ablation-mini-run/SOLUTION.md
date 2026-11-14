# Exercise 05 — DataComp-LM Style Ablation Mini Run (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner runs a small-scale
DataComp-LM–style ablation: a fixed training recipe is trained twice (or
three times) on competing *data mixes* built from the mod-103 slice, and
the resulting models are scored on a fixed held-out benchmark set. The
goal is to internalise the DCLM experimental design:

- **Training compute and model are held constant.**
- **Only the data mix varies between runs.**
- **Each run writes a run card** (data mix spec, filter configs, model
  config, tokenizer, seed, compute budget, eval scores).
- **Pairwise deltas on eval scores are what's reported,** not absolute
  scores on the mini run (which are dominated by the tiny compute
  budget and are not transferable).

This is a *mini* run: a few million tokens, a small model, a single
GPU, running in minutes-to-a-few-hours on consumer hardware. The point
is the **methodology** — reproducible run cards, honest pairwise deltas,
scope-limited claims — not state-of-the-art numbers.

A good solution produces:

1. Two (or three) **data mixes** differing by exactly one operator
   choice from mod-103 (e.g. language threshold 0.5 vs 0.7, or with vs
   without an extra quality filter).
2. A **training script** parameterised only by the mix id; everything
   else (model, tokenizer, optimiser, schedule, seed) is a fixed
   constant.
3. A **run card** per run with the data-mix manifest SHA, trainer
   config SHA, git SHA, tokenizer SHA, hardware, wall-clock, final
   train loss, and eval scores.
4. A **comparison report** that prints the pairwise deltas, confidence
   intervals (from multi-seed runs if compute allows), and an explicit
   "this is not a DCLM leaderboard entry" disclaimer.

Scope framing: DataComp-LM (DCLM) is a benchmark *framework* for
ranking data pipelines; the paper's headline numbers depend on specific
compute budgets, specific baseline model configs, and specific eval
suites. A learner's mini run cannot reproduce those numbers, and
reporting them would be dishonest. The exercise is scaffolded to the
methodology *only*. Re-confirm DCLM's exact baseline model sizes and
compute budgets against the current paper / repo before publishing;
those evolve.

Governance framing: ablations are how we convert "does this filter
help?" from a hunch into a measurement. NIST AI RMF `MEASURE 2.3`
(system performance measured) and `MEASURE 2.11` (bias examined) both
expect data-side decisions to be defended with evidence; an ablation
run card is the evidence.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── requirements.lock
├── Makefile
├── mixes/
│   ├── mix-a.yaml                # e.g. lang_score ≥ 0.5
│   ├── mix-b.yaml                # e.g. lang_score ≥ 0.7
│   └── mix-c.yaml                # optional: mix-a + a Dolma gopher filter
├── eval/
│   ├── eval_suite.yaml           # fixed eval tasks + prompts
│   └── held_out.jsonl.gz
├── runs/
│   └── run_YYYYMMDD_HHMM_<mix>/  # one directory per run
│       ├── run_card.json
│       ├── tokenizer.json        # pinned copy (or path + SHA)
│       ├── training.log
│       └── model.safetensors
├── code/
│   ├── 01_build_mix.py
│   ├── 02_train.py
│   ├── 03_eval.py
│   └── 04_compare.py
└── report/
    └── ablation_report.md
```

### 2.2 `mixes/mix-a.yaml` — define the data mix, pin everything

```yaml
# mixes/mix-a.yaml
mix_id: "mix-a"
slice_name: "slice-cc-en-small-v1"
slice_version: "1.0.0"
slice_manifest_sha256: "<sha of ../mod-103/manifest/slice_manifest.json>"
filters:
  - name: language_filter
    params: {column: language, equals: en}
  - name: language_score_threshold
    params: {column: language_score, min: 0.5}
tokenizer:
  name: "<pin a tokenizer, e.g. gpt2 / tiktoken gpt2 / a hf tokenizer>"
  sha256: "<tokenizer file sha256>"
target_tokens: 50_000_000     # mini run; keep modest
pack:
  seq_length: 1024
  seed: 20260101
```

The file is the single source of truth: `01_build_mix.py` reads it,
applies the listed filters, tokenizes, packs, and writes the per-token
shard to disk, hashed. `mix-b.yaml` differs only in
`language_score_threshold.min = 0.7`. `mix-c.yaml` is the
one-extra-operator mix.

### 2.3 `code/01_build_mix.py`

```python
# code/01_build_mix.py
# Read mix yaml, apply filters against mod-103's per-record manifest,
# tokenize + pack. Output: a token shard + a mix manifest.
from __future__ import annotations

import gzip
import hashlib
import json
import sys
from pathlib import Path

import numpy as np
import polars as pl
import yaml
from tokenizers import Tokenizer  # or transformers.AutoTokenizer

SRC = Path("../mod-103/output/slice-00000.jsonl.gz")  # from mod-103 Exercise 01


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def apply_filters(docs: list[dict], per_record: pl.DataFrame,
                  filters: list[dict]) -> list[dict]:
    manifest_by_wid = {row["warc_record_id"]: row for row in per_record.iter_rows(named=True)}
    out = []
    for d in docs:
        wid = d.get("warc_record_id", "")
        meta = manifest_by_wid.get(wid, {})
        ok = True
        for f in filters:
            name, p = f["name"], f["params"]
            if name == "language_filter":
                ok = ok and meta.get(p["column"]) == p["equals"]
            elif name == "language_score_threshold":
                ok = ok and (meta.get(p["column"], 0.0) or 0.0) >= p["min"]
            else:
                raise ValueError(f"unknown filter: {name}")
        if ok:
            out.append(d)
    return out


def main(mix_path: str) -> None:
    mix = yaml.safe_load(Path(mix_path).read_text())
    per_record = pl.read_parquet(
        "../mod-103/manifest/per_record_manifest.parquet"
    )
    docs = [json.loads(l) for l in gzip.open(SRC, "rt", encoding="utf-8")]
    kept = apply_filters(docs, per_record, mix["filters"])

    tok = Tokenizer.from_file(mix["tokenizer"]["sha256"]
                              if Path(mix["tokenizer"]["sha256"]).exists()
                              else mix["tokenizer"]["name"])
    rng = np.random.default_rng(mix["pack"]["seed"])

    target_tokens = int(mix["target_tokens"])
    seq_len = int(mix["pack"]["seq_length"])
    stream: list[int] = []
    rng.shuffle(kept)
    for d in kept:
        ids = tok.encode(d["text"]).ids
        stream.extend(ids + [tok.token_to_id("<|endoftext|>") or 0])
        if len(stream) >= target_tokens:
            break
    stream = stream[:target_tokens]
    arr = np.array(stream, dtype=np.uint32)
    # pack into (seq_len)-long sequences; drop remainder
    n_seq = len(arr) // seq_len
    packed = arr[:n_seq * seq_len].reshape(n_seq, seq_len)

    out_dir = Path(f"mixes/_out_{mix['mix_id']}")
    out_dir.mkdir(parents=True, exist_ok=True)
    packed.tofile(out_dir / "tokens.bin")

    manifest = {
        "mix_id": mix["mix_id"],
        "mix_yaml_sha256": sha256_file(Path(mix_path)),
        "slice_manifest_sha256": mix["slice_manifest_sha256"],
        "tokenizer_sha256": mix["tokenizer"]["sha256"],
        "kept_docs": len(kept),
        "produced_tokens": int(n_seq * seq_len),
        "tokens_bin_sha256": sha256_file(out_dir / "tokens.bin"),
    }
    (out_dir / "mix_manifest.json").write_text(json.dumps(manifest, indent=2, sort_keys=True))
    print(json.dumps(manifest, indent=2))


if __name__ == "__main__":
    main(sys.argv[1])
```

### 2.4 `code/02_train.py` — fixed trainer, only the mix varies

```python
# code/02_train.py
# One run, one mix. Everything except the mix is fixed in this file:
# model config, optimizer, schedule, seed, context length, loss.
# This is the whole point of a DCLM-style ablation.
from __future__ import annotations

import datetime as dt
import hashlib
import json
import sys
import time
from pathlib import Path

import numpy as np
import torch
from torch import nn

SEQ_LEN = 1024
BATCH_SIZE = 8
STEPS = 2000           # mini run
LR = 3e-4
SEED = 20260101
MODEL = {"n_layer": 4, "n_head": 4, "d_model": 256, "vocab_size": 50257}

def set_seed(seed: int) -> None:
    import random
    random.seed(seed); np.random.seed(seed); torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)

class TinyDecoder(nn.Module):
    # Keep it minimal: one small transformer decoder stack.
    # In practice, swap for a known-reference implementation and pin its SHA.
    def __init__(self, cfg: dict) -> None:
        super().__init__()
        layer = nn.TransformerDecoderLayer(
            d_model=cfg["d_model"], nhead=cfg["n_head"], batch_first=True,
            dim_feedforward=cfg["d_model"] * 4,
        )
        self.emb = nn.Embedding(cfg["vocab_size"], cfg["d_model"])
        self.stack = nn.TransformerDecoder(layer, num_layers=cfg["n_layer"])
        self.head = nn.Linear(cfg["d_model"], cfg["vocab_size"], bias=False)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        h = self.emb(x)
        causal = nn.Transformer.generate_square_subsequent_mask(x.size(1)).to(x.device)
        return self.head(self.stack(h, h, tgt_mask=causal))


def load_shards(mix_id: str) -> np.ndarray:
    data = np.fromfile(f"mixes/_out_{mix_id}/tokens.bin", dtype=np.uint32)
    return data.reshape(-1, SEQ_LEN)


def main(mix_id: str) -> None:
    set_seed(SEED)
    device = "cuda" if torch.cuda.is_available() else "cpu"
    data = load_shards(mix_id)

    model = TinyDecoder(MODEL).to(device)
    opt = torch.optim.AdamW(model.parameters(), lr=LR, betas=(0.9, 0.95))
    losses = []

    t0 = time.time()
    for step in range(STEPS):
        idx = np.random.randint(0, data.shape[0], BATCH_SIZE)
        batch = torch.tensor(data[idx].astype(np.int64), device=device)
        logits = model(batch)
        loss = nn.functional.cross_entropy(
            logits[:, :-1].reshape(-1, MODEL["vocab_size"]),
            batch[:, 1:].reshape(-1),
        )
        opt.zero_grad(); loss.backward(); opt.step()
        losses.append(float(loss))
        if step % 100 == 0:
            print(f"[train] step {step}  loss {float(loss):.4f}")
    wall = time.time() - t0

    stamp = dt.datetime.utcnow().strftime("%Y%m%d_%H%M")
    run_dir = Path(f"runs/run_{stamp}_{mix_id}")
    run_dir.mkdir(parents=True, exist_ok=True)
    torch.save(model.state_dict(), run_dir / "model.pt")
    (run_dir / "run_card.json").write_text(json.dumps({
        "mix_id": mix_id,
        "mix_manifest": json.loads(
            Path(f"mixes/_out_{mix_id}/mix_manifest.json").read_text()),
        "model": MODEL,
        "optim": {"name": "AdamW", "lr": LR, "betas": [0.9, 0.95]},
        "steps": STEPS,
        "batch_size": BATCH_SIZE,
        "seq_len": SEQ_LEN,
        "seed": SEED,
        "device": device,
        "wall_seconds": wall,
        "final_train_loss": losses[-1],
        "train_loss_history_sha256": hashlib.sha256(
            json.dumps(losses).encode()).hexdigest(),
    }, indent=2, sort_keys=True))


if __name__ == "__main__":
    main(sys.argv[1])
```

Scope caveat: DCLM's reference baseline uses OpenLM / specific model
sizes (e.g. 7B, 1B, 400M) and a specific compute budget. The mini model
above is **not** a DCLM baseline; the run card must explicitly say so.

### 2.5 `code/03_eval.py` — fixed eval suite

```python
# code/03_eval.py
# Score the model on a FIXED small held-out set. The set and prompts live
# in eval/eval_suite.yaml — never edit between runs.
from __future__ import annotations

import gzip
import json
import math
import sys
from pathlib import Path

import numpy as np
import torch
import yaml

from code02_train_module import TinyDecoder, MODEL, SEQ_LEN  # same model config


def main(run_dir: str) -> None:
    suite = yaml.safe_load(Path("eval/eval_suite.yaml").read_text())
    device = "cuda" if torch.cuda.is_available() else "cpu"

    model = TinyDecoder(MODEL).to(device)
    model.load_state_dict(torch.load(Path(run_dir) / "model.pt",
                                     map_location=device))
    model.eval()

    # Perplexity on a held-out slice of text.
    ids_stream: list[int] = []
    with gzip.open(suite["held_out"], "rt", encoding="utf-8") as fh:
        from tokenizers import Tokenizer
        tok = Tokenizer.from_file(suite["tokenizer"])
        for line in fh:
            d = json.loads(line)
            ids_stream.extend(tok.encode(d["text"]).ids)
            if len(ids_stream) > 200_000:
                break
    arr = torch.tensor(ids_stream[: (len(ids_stream) // SEQ_LEN) * SEQ_LEN],
                       dtype=torch.long, device=device).reshape(-1, SEQ_LEN)

    with torch.no_grad():
        total_loss, total_tokens = 0.0, 0
        for i in range(0, arr.size(0), 8):
            batch = arr[i:i+8]
            logits = model(batch)
            loss = torch.nn.functional.cross_entropy(
                logits[:, :-1].reshape(-1, MODEL["vocab_size"]),
                batch[:, 1:].reshape(-1), reduction="sum",
            )
            total_loss += float(loss)
            total_tokens += batch.numel() - batch.size(0)
    ppl = math.exp(total_loss / total_tokens)

    eval_card = {
        "eval_suite_sha256": __import__("hashlib").sha256(
            Path("eval/eval_suite.yaml").read_bytes()).hexdigest(),
        "held_out_sha256": __import__("hashlib").sha256(
            Path(suite["held_out"]).read_bytes()).hexdigest(),
        "perplexity": ppl,
        "eval_tokens": total_tokens,
    }
    (Path(run_dir) / "eval_card.json").write_text(
        json.dumps(eval_card, indent=2, sort_keys=True))
    print(json.dumps(eval_card, indent=2))


if __name__ == "__main__":
    main(sys.argv[1])
```

Perplexity is one metric. A real DCLM-style ablation uses a basket of
downstream tasks; add any you want (e.g. small zero-shot classifiers,
toy multiple-choice) **as long as they are listed in
`eval/eval_suite.yaml` and never edited between runs**.

### 2.6 `code/04_compare.py` — the pairwise report

```python
# code/04_compare.py
# Aggregate per-run eval cards, compute deltas vs the baseline mix,
# emit a compact Markdown comparison report.
from __future__ import annotations

import json
from pathlib import Path

import polars as pl


def main() -> None:
    rows = []
    for run_dir in sorted(Path("runs").glob("run_*")):
        rc = json.loads((run_dir / "run_card.json").read_text())
        ec = json.loads((run_dir / "eval_card.json").read_text())
        rows.append({
            "run": run_dir.name,
            "mix_id": rc["mix_id"],
            "final_train_loss": rc["final_train_loss"],
            "perplexity": ec["perplexity"],
            "wall_seconds": rc["wall_seconds"],
            "mix_manifest_sha256": rc["mix_manifest"]["tokens_bin_sha256"],
        })
    df = pl.DataFrame(rows)
    baseline = df.filter(pl.col("mix_id") == "mix-a")
    if len(baseline) == 0:
        raise SystemExit("mix-a is the baseline; expected at least one run")
    base_ppl = float(baseline[0, "perplexity"])
    df = df.with_columns(
        (pl.col("perplexity") - base_ppl).alias("delta_ppl_vs_mix_a")
    )
    Path("report").mkdir(exist_ok=True)
    Path("report/ablation_report.md").write_text(
        "# DCLM-style mini ablation\n\n"
        "**This is NOT a DCLM leaderboard entry.** Compute budget and model "
        "size are an order of magnitude below DCLM's baselines; only pairwise "
        "deltas between mixes trained in this notebook are interpretable.\n\n"
        + df.to_pandas().to_markdown(index=False)
    )
    print(df.to_pandas().to_markdown(index=False))


if __name__ == "__main__":
    main()
```

### 2.7 `README.md` (excerpts the reviewer needs)

Record, with evidence:

- Hardware (GPU model, driver, torch + cuda versions).
- Tokenizer SHA.
- Model config SHA (the dict in `02_train.py`).
- Eval suite SHA.
- Seeds for data packing + training.
- The **not-a-DCLM-entry disclaimer** — repeated verbatim in the report.

### 2.8 Where the limits are honest

The mini run **does not** tell you:

- Whether a filter choice helps at scale. Tiny models are noisy and
  non-monotone in data-quality signals; DCLM's own methodology uses a
  specific baseline model + compute budget precisely for this reason.
- Whether a filter choice generalises across evals. Perplexity on a
  held-out slice is one scalar; real DCLM uses a suite of downstream
  tasks.
- Whether the training recipe is optimal. Everything except the data
  mix is a constant by design; recipe ablations are out of scope.

If you cannot run the full DCLM baseline, do not claim DCLM results —
state the limits of the mini run and move on.

## 3. Validation steps

1. **Only the mix varies.** Diff `mix-a.yaml` vs `mix-b.yaml`; exactly
   one operator parameter should differ. Everything in `02_train.py`
   (model, optimiser, seed, steps) is a constant.
2. **Every run has a run card** with mix manifest SHA, tokenizer SHA,
   eval suite SHA, seeds, wall-clock, final train loss, eval scores.
   Missing any field is a review failure.
3. **The eval suite is frozen.** `SHA256(eval/eval_suite.yaml)` is the
   same across runs. If it differs, the comparison is invalid.
4. **The comparison report prints pairwise deltas, not absolutes-as-
   SOTA.** The DCLM-disclaimer must be present verbatim.
5. **Reproducibility.** Running `make all` twice on the same mix (same
   seed, same tokenizer, same hardware) produces the same final loss to
   at least 3 decimals. If it doesn't, something isn't seeded.
6. **The slice input is pinned.** Every mix manifest records
   `slice_manifest_sha256` from mod-103; a slice rebuild invalidates
   the ablation.
7. **No DCLM benchmark numbers invented.** No row in the report
   compares against DCLM's published scores unless the mini run was
   *actually* run against DCLM's baseline model + compute, which it
   wasn't.
8. **All unverifiable DCLM claims flagged.** Anything that would need
   the paper to confirm is called out in prose as "verify against paper"
   or wrapped in `<!-- unverified: ... -->`; no silent guesses.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
A passing solution scores ≥ 20/28 and loses zero points on rows marked
**critical**.

| # | Dimension                                                                                            | Critical? |
|---|------------------------------------------------------------------------------------------------------|-----------|
| 1 | At least 2 mixes; mixes differ by exactly one operator parameter                                     | Yes       |
| 2 | Training recipe (model + optimiser + schedule + seed) is identical across runs                       | Yes       |
| 3 | Each run has a run card with every field from §2.4                                                   | Yes       |
| 4 | Eval suite is frozen and SHA-pinned                                                                  | Yes       |
| 5 | Report prints pairwise deltas, not absolute "ranks"                                                  | Yes       |
| 6 | Not-a-DCLM-entry disclaimer present verbatim in report and README                                    | Yes       |
| 7 | Slice input pinned by SHA back to mod-103                                                            | Yes       |
| 8 | Reproducibility: same mix + seed + hardware → same loss to 3 decimals                                | Yes       |
| 9 | Hardware + torch/cuda versions recorded in run card                                                  | No        |
| 10 | Governance anchor cited (NIST AI RMF `MEASURE 2.3` / OWASP ML Top 10)                               | No        |
| 11 | Unverified DCLM facts are flagged explicitly (prose caveat or `<!-- unverified: ... -->`) instead of guessed | Yes       |
| 12 | Report includes a "limits of inference" paragraph                                                    | Yes       |
| 13 | `Makefile` reproduces the full ablation from a clean clone (end-to-end, both mixes)                  | No        |
| 14 | Perplexity (or whatever metric) reports eval token count alongside — not a bare number               | No        |

## 5. Common mistakes

- **Comparing absolute perplexity to DCLM's published numbers.** You are
  training a tiny model on a tiny budget. Absolute numbers aren't
  transferable; delta-between-your-own-mixes is the only interpretable
  comparison.
- **Changing more than one thing between mixes.** "Mix B has threshold
  0.7 and uses a different tokenizer" is not an ablation; it's two
  things at once and the result is uninterpretable.
- **Editing the eval set between runs.** Even by accident. Freeze the
  JSONL + the suite YAML and pin SHAs. Any change to either requires a
  new ablation.
- **Not seeding packing.** If `01_build_mix.py` doesn't seed the shuffle,
  the mix is non-reproducible, which means the ablation is
  non-reproducible.
- **Reporting train loss as "the result".** Train loss is noisy and
  easily affected by data ordering. Perplexity on a held-out set (or a
  task score on a frozen eval) is the dependent variable.
- **No "limits of inference" section.** DCLM's paper is itself careful
  about what its scale-wise conclusions apply to; a mini run has an
  even smaller envelope. Write it down.
- **Running only once per mix.** Variance across seeds at this model
  size is real. If compute allows, run each mix with 2–3 seeds and
  report a mean ± sample stddev instead of a point estimate. If not,
  mark the result "single-seed, interpretation gated on variance
  unknown".
- **Dropping the slice SHA.** Six months later nobody will know which
  slice the ablation ran against. The mix manifest captures the slice
  SHA precisely to defend against this.

## 6. References

- **DataComp-LM (DCLM).** The paper + benchmark framework this exercise
  imitates methodologically. Pin whichever paper version you read:
  <https://arxiv.org/abs/2406.11794>. The DCLM repo:
  <https://github.com/mlfoundations/dclm>. Re-confirm the arXiv ID and
  repo URL at write time rather than trusting this template.
- **OpenLM** — the reference open-source LM training stack DCLM builds
  on. <https://github.com/mlfoundations/open_lm>. Re-confirm that this is
  still the DCLM-recommended trainer before citing it.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MEASURE 2.3`
  ("system performance measured") grounds the ablation as the evidence
  for a data-side decision.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02 (Data Poisoning) is why "which filter helps?" is a
  security-relevant, not just performance-relevant, question.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. "Poison Training Data"
  framing for why filter choices matter beyond eval scores.
- **Local anchors:** mod-103 Exercise 01 (the slice both mixes draw
  from); Exercise 01 of this module (the slice's quality bar);
  Exercise 04 of this module (subpopulation audit — complementary;
  ablation proves *helps / hurts*, subpop audit proves *representative
  / skewed*); mod-106 (contamination — the eval suite's held-out set
  must be contamination-checked).
