# judge-eval pack

Templates for mizan Experiment 07 (`docs/experiments/07-judge-capability-rerun.md`) and Experiment 08
(`docs/experiments/08-gemini-4-hydrogen-spike.md` in the mizan repo), which score autoregressive (Gemini),
non-autoregressive (DiffusionGemma) and no-model judges against human or gold labels across the Vertex Gen AI
evaluation capabilities.

| Group | Templates | Kind | Default Model | Gold source |
|---|---|---|---|---|
| Binary judges (fast) | `response-harm-check`, `answer-faithfulness-check` | `boul` (`passed` = harmful / faithful) | `gemini-3.5-flash-lite` | BeaverTails, HaluBench |
| Binary judge (nuanced toxicity) | `prompt-toxicity-check` | `boul` (detector polarity: `passed` = toxic) | `gemini-4-hydrogen` | ToxicChat |
| Likert | `helpfulness-score` (0–4), `summary-coherence-score` (1–5) | `score` with `rubricDetail.scale` so every engine answers on the same scale | `gemini-4-hydrogen` | HelpSteer2, SummEval |
| Pairwise | `pairwise-response-quality`, `pairwise-multiturn-quality`, `pairwise-instruction-following` | `pairwise` (BASELINE = first, CANDIDATE = second) | `gemini-4-hydrogen` | MT-Bench human judgments, RewardBench/LLMBar |
| Vertex predefined | `prebuilt-safety`, `prebuilt-groundedness`, `prebuilt-fluency`, `prebuilt-coherence` | `prebuilt` (service judge; `autorater` is ignored) | service-managed | as above |
| No model | `exact-match`, `bleu`, `rouge-lsum`, `tool-*`, `trajectory-*` | `computation` | none (local Go / Vertex) | sacrebleu / rouge_score / known values |

`prebuilt` and `computation` need mizan with `spec.native` support (mizan PR adding Experiment 07). Build the
datasets with mizan's `scripts/judge-eval/build_suites.py`; several upstream licences are non-commercial, so the
datasets are not stored here.
