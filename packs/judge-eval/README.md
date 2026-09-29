# judge-eval pack

Templates for mizan Experiment 07 (`docs/experiments/07-judge-capability-rerun.md` in the mizan repo), which
scores autoregressive (Gemini), non-autoregressive (DiffusionGemma) and no-model judges against human or gold
labels across the Vertex Gen AI evaluation capabilities.

| Group | Templates | Kind | Gold source |
|---|---|---|---|
| Binary judges | `response-harm-check`, `prompt-toxicity-check`, `answer-faithfulness-check` | `boul` (detector polarity: `passed` = condition holds, except faithfulness where `passed` = faithful) | BeaverTails, ToxicChat, HaluBench |
| Likert | `helpfulness-score` (0–4), `summary-coherence-score` (1–5) | `score` with `rubricDetail.scale` so every engine answers on the same scale | HelpSteer2, SummEval |
| Pairwise | `pairwise-response-quality`, `pairwise-multiturn-quality`, `pairwise-instruction-following` | `pairwise` (BASELINE = first, CANDIDATE = second) | MT-Bench human judgments, RewardBench/LLMBar |
| Vertex predefined | `prebuilt-safety`, `prebuilt-groundedness`, `prebuilt-fluency`, `prebuilt-coherence` | `prebuilt` (service judge; `autorater` is ignored) | as above |
| No model | `exact-match`, `bleu`, `rouge-lsum`, `tool-*`, `trajectory-*` | `computation` | sacrebleu / rouge_score / known values |

`prebuilt` and `computation` need mizan with `spec.native` support (mizan PR adding Experiment 07). Build the
datasets with mizan's `scripts/judge-eval/build_suites.py`; several upstream licences are non-commercial, so the
datasets are not stored here.
