# calibration — example cases

Three case files, one per decision primitive, in the JSONL schema that
`mizan eval compare-engines --dataset` reads:

```json
{"id": "...", "metric": "calibration/...", "tier": "...", "category": "...",
 "fields": {"<input>": "<value>"}, "expected": "..."}
```

| File | Cases | Primitives shown |
|---|---|---|
| [`boul-cases.jsonl`](boul-cases.jsonl) | 16 | toxicity, passage relevance, prompt injection, agent step hijack, grounding, BoolQ |
| [`choice-cases.jsonl`](choice-cases.jsonl) | 20 | emotion, intent + out-of-scope, NLI, soft NLI, agent step label, BANKING77 |
| [`score-cases.jsonl`](score-cases.jsonl) | 10 | toxicity fraction, Yelp stars, SST-5 levels |

```bash
mizan eval compare-engines \
  --dataset packs/calibration/examples/boul-cases.jsonl \
  --engine-a vertex --engine-b vertex \
  --model-a gemini-2.5-flash --model-b gemini-2.5-pro \
  --output-file report.json
```

> [!IMPORTANT]
> **These rows are hand-written, not sampled from the datasets.** They are written
> in the style of each corpus — including the ambiguous and adversarial cases that
> make it interesting — so the files are safe to redistribute under this repo's
> Apache-2.0 licence. Several of the source datasets are non-commercial, no-derivatives,
> or share-alike (see the [pack README](../README.md#dataset-access-status)), so their
> real rows must not be committed here. To benchmark for real, materialise rows locally
> with the recipes in
> [Building real case files](../README.md#building-real-case-files-from-the-source-datasets)
> and keep the output out of git.

> [!NOTE]
> `expected` is carried into the report but **never scored by Mizan** — `--dataset`
> measures engine-A-vs-engine-B agreement. See
> [Running a template against gold labels](../README.md#running-a-template-against-gold-labels)
> for the one-line `jq` pass that turns the report into an accuracy number.

## Field conventions

- **boul** — `expected` is the string `"true"` or `"false"`, matching the JSON
  boolean `passed`. In this pack `true` always means *the proposition holds /
  the thing was detected*, never *this artifact is fine*.
- **choice** — `expected` is the selection string, byte-for-byte identical to an
  entry in the template's `choices`.
- **score** — `expected` is the gold number as a string. Yelp is 1–5 (gold
  `label + 1`); SST-5 is 0–4 (gold `label`, no offset); the toxicity rate is a
  0.0–1.0 rater fraction scored by mean absolute error, not exact match.

`tier` drives the per-tier breakdown in the report. The tiers used here are
`easy`, `ambiguous`, `adversarial`, `out-of-scope`, `localization`,
`low-entropy` / `high-entropy`, and `held-out` — a judge that only holds up on
`easy` is the exact result this pack exists to surface.
