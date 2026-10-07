# longform-video pack

Gemini LLM-as-a-Judge templates for evaluating **coherent long-form / multi-shot
genmedia video**.

These are **inspired-by teaching templates**, not faithful reproductions of any
paper. They are modeled on the four frameworks described in Google Research's blog
[*Automating coherent long-form video generation*](https://research.google/blog/coherent-long-form-video-generation/)
— **Co-Director**, **CANVAS**, **A²RD**, and **VQQA** — each of which places an
MLLM/VLM judge at its core. That judge role is exactly what Mizan is, so these
templates realize it over the `video` modality.

> The research papers (COLM/EMNLP 2026) are not public; everything here is derived
> from the blog's technique descriptions.

## Templates

| id | kind | measures | framework anchor |
|---|---|---|---|
| `longform-video/cross-shot-consistency` | `score` 1-5 | character / costume / prop / location continuity between two shots, incl. non-consecutive reappearance | CANVAS |
| `longform-video/narrative-coherence` | `score` 1-5 | whether a stitched multi-segment video tells one continuous, non-contradictory story (no content collapse, no uncontrolled drift) | A²RD |
| `longform-video/prompt-adherence` | `pointwise` | faithfulness of the video to the original, unedited prompt (anti-drift global rater) | VQQA (Global Selection) |
| `longform-video/overall-coherence-score` | `score` 1-5 | overall coherence of the whole piece (video analogue of `judge-eval/summary-coherence-score`) | general / calibration anchor |
| `longform-video/factored-quality-scorecard` | `rubric` | per-criterion factored reward: narrative progression, visual consistency, prompt adherence, technical quality | Co-Director (factored reward) |
| `longform-video/continuity-pairwise` | `pairwise` | which of two renders better preserves continuity against a contract | CANVAS A/B + VQQA Global Selection |
| `longform-video/vqqa-visual-questions` | `custom_schema` | structured visual-question critique with a `suggested_prompt_fix` per defect (the "semantic gradient") | VQQA (semantic gradients) |

## Inputs & assets

Video inputs are supplied at eval time as `gs://` URIs (`--gcs key=gs://…`) or
local files auto-staged to GCS (`--file key=/path`). The pack carries no media; see
`examples/` for the field shapes.

**Long-form note:** whether the Vertex Eval autorater ingests a full multi-minute
video or many shots in one instance is not documented. Prefer **segment-level** or
**shot-pair** inputs (e.g. `cross-shot-consistency` takes two shots; `*-pairwise`
takes two renders). Treat whole-sequence templates (`narrative-coherence`,
`overall-coherence-score`) as applicable at the length the autorater accepts.

## On NIQE (no-reference perceptual quality)

NIQE and other **no-reference / blind** perceptual metrics are **computed** signals
(natural-scene statistics), not LLM-as-judge prompts. The Mizan template format
cannot express a per-frame pixel-statistics computation (the only non-LLM `kind`,
`heuristic`, does string checks — `contains/regex/equals/json-valid/json-schema-valid`;
`native`/`computation` covers only Vertex text metrics). So there is **no** NIQE
template here by design. The blog itself names no no-reference metric; it relies on
MLLM/VLM judges. If you want NIQE, compute it in your agent (e.g. `pyiqa`) and either
(a) pass the number into `factored-quality-scorecard` as part of the brief for the
judge to weigh, or (b) use it as a cheap pre-filter gate before spending judge
credits. See the pack PR / deliverable for the full analysis.

## Validate

```
mizan pack validate packs/longform-video
```
