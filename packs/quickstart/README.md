# quickstart pack

**The recommended first pack to import.** It is a runnable smoke-test of your
Vertex AI project, ADC credentials, and model setup: import it, run a couple of
cells, and confirm mizan can author a template, reach the Eval Service, and
score every metric kind and modality end-to-end. Every template works
out-of-the-box on a fresh install — no extra configuration beyond a project,
location, and staging bucket.

- **Namespace:** `quickstart` (every template id begins `quickstart/`)
- **Maintainers:** `mizan-quickstart-team`
- **License:** Apache-2.0
- **Default judge:** `gemini-2.5-flash` (mizan's built-in default)

```bash
mizan registry import <path-to-mizan-templates-checkout>
mizan eval run --metric quickstart/text-response-helpfulness \
  --field response="I've resent your activation link; it should arrive in 2 minutes."
```

## Templates

Ten templates spanning **all four metric kinds** × **four modalities** — that
spread is the point: one import exercises every path mizan can take.

| id | kind | modalities | what it does |
|---|---|---|---|
| `quickstart/text-response-helpfulness` | pointwise | text | Score a single customer-support reply for helpfulness on an explicit 1–5 scale. |
| `quickstart/image-visual-quality` | pointwise | image | Score the overall visual quality of an image on a 1–5 scale. |
| `quickstart/video-brand-alignment` | pointwise | video, text | Score how well a video ad aligns with a supplied brand guideline. |
| `quickstart/audio-narration-clarity` | pointwise | audio | Rate the clarity and intelligibility of spoken-word narration on a 1–5 scale. |
| `quickstart/text-answer-comparison` | pairwise | text | Compare two text answers and pick the more helpful, accurate one. |
| `quickstart/video-pairwise-comparison` | pairwise | video | Compare two video ads end-to-end and pick the more compelling one. |
| `quickstart/image-rubric-scorecard` | rubric | image | Score an image against a multi-criterion rubric with per-criterion scoring. |
| `quickstart/text-rubric-scorecard` | rubric | text | Score a technical answer against a multi-criterion rubric with a per-criterion breakdown. |
| `quickstart/text-structured-extraction` | custom_schema | text | Extract structured triage fields from a support message per a JSON schema. |
| `quickstart/audio-structured-extraction` | custom_schema | audio | Extract a structured order from spoken audio per a JSON schema. |

### What each metric kind demonstrates

- **pointwise** — score one response against a scale. The simplest kind; the
  four pointwise cells show the same authoring pattern working across text,
  image, video, and audio.
- **pairwise** — present two responses and pick the better one. Use `Choice` as
  the authoritative verdict. Works for text and, via `--gcs`/`--file`, for media.
- **rubric** — score against a set of named criteria. Run natively for a single
  overall score, or with `--rubric-detail` for a per-criterion breakdown.
- **custom_schema** — extract a strict, machine-parseable JSON object you define.
  The schema's field descriptions are the contract for what the judge returns.

### What each modality demonstrates

- **text** — inline responses via `--field`; the fastest path to a first score.
- **image / video / audio** — supplied as `gs://` URIs via `--gcs` (or local
  files via `--file`). Vertex reads public buckets directly, so no copy to your
  staging bucket is needed for the sample assets these cells use.

## Try a multimodal cell

The multimodal cells reference Google's **public** generative-AI sample bucket
(`gs://cloud-samples-data/generative-ai/...`), readable by Vertex directly, so
they run without you staging any assets:

```bash
# rubric with a per-criterion breakdown that genuinely reads the image
mizan eval run --metric quickstart/image-rubric-scorecard \
  --gcs image=gs://cloud-samples-data/generative-ai/image/320px-Felis_catus-cat_on_snow.jpg \
  --rubric-detail --rubric-scale 1-5

# structured extraction from real spoken audio
mizan eval run --metric quickstart/audio-structured-extraction \
  --gcs order_audio=gs://cloud-samples-data/generative-ai/audio/coffee_order.wav

# media pairwise: compare two real video ads end-to-end
mizan eval pairwise --metric quickstart/video-pairwise-comparison \
  --gcs baseline_ad=gs://cloud-samples-data/generative-ai/video/google_home_celebrity_ad.mp4 \
  --gcs candidate_ad=gs://cloud-samples-data/generative-ai/video/ad_copy_from_video.mp4
```

Each is a positive **multimodal-correctness** check: the per-criterion image
rubric scores brand-logo criteria low because there is no logo in the photo; the
audio extraction returns real order contents with `transcript_heard: true`; and
the video pairwise describes each clip's actual content — all confirming the
media genuinely reaches the judge, not just the file path.

## Why these templates are written this way

These conventions come out of a live authoring exploration across all four kinds
and modalities; each template here follows them, and they are the lessons to
carry into your own templates.

1. **State the scoring scale explicitly and anchor both ends semantically**
   (`1 = …, 5 = …`). This is the single cheapest, strongest calibration lever —
   see the pointwise cells.
2. **Make rubric criteria atomic and independently checkable** — one criterion =
   one fact. Compound criteria joined by "and" cannot split cleanly; separate
   criteria (e.g. "mentions indexing" vs "mentions EXPLAIN") discriminate crisply.
3. **Let custom_schema field descriptions carry the contract** — enumerate the
   allowed values in the description (`"one of: billing, technical, account,
   other"`) and name fields for the consumer's action (`action_items`,
   `pii_present`). Description quality drives output quality.
4. **Give multimodal structured templates a provenance/canary field** (e.g.
   `transcript_heard`, `asset_received`). It costs one boolean and turns "did the
   asset actually reach the judge?" into a value you can read back — cheap
   insurance and a clear positive signal that the media path is working.
5. **Pick the path that matches the output you want.** Native pointwise or native
   rubric gives a single overall score; the genai structured path (custom_schema,
   or rubric with `--rubric-detail`) gives per-field / per-criterion breakdowns.
   Both attach multimodal assets correctly, so choose by the shape of result you
   need, not by modality.
6. **Provide reference material as its own text input**, not baked into the
   prompt. `quickstart/video-brand-alignment` takes the brand guideline as
   `{{brand_guideline}}`, so the same template is reusable across guidelines.
7. **Use `systemInstruction` to set severity or stance** ("Be strict…")
   separately from the task prompt, so the two can be tuned independently.
8. **Trust the pairwise `Choice`, not the prose labels.** Position-bias flipping
   can scramble which side the explanation calls "baseline" vs "candidate"; the
   `Choice` is the de-biased, authoritative verdict. Don't parse baseline/candidate
   identity out of the explanation text.

## A note on calibration (prose, not a checked value)

Scores are model behavior and drift over time, so this pack carries template
*definitions* only — no expected results are baked into the YAML. As a rough
calibration reference from a live run on `gemini-2.5-flash`: a clearly good
support reply scored 5 and a dismissive one scored 1; the cat-on-snow photo
scored 5 for visual quality; and the image rubric scored brand-logo criteria 1
(correctly — there is no logo) while safety and technical quality scored 5.
Treat these as illustrative, not as a contract.

### Try this next

Re-run any cell with `--model gemini-3.5-flash-lite` to see a **stricter, more
literal** calibration: on guideline- and rubric-anchored multimodal cells the
newer model tends to score lower and enforce criteria more literally (for
example, penalizing competitor-brand mentions a `gemini-2.5-flash` judge may
gloss). If you tuned prompts or thresholds against `gemini-2.5-flash`, re-check
them before switching defaults. This is an aside, not a requirement — the pack
runs on its `gemini-2.5-flash` default out of the box.

## Format

See the repository [`docs/pack-format.md`](../../docs/pack-format.md) for the
pack and template file format. These templates were authored with
`mizan registry create`, exported with `mizan registry export`, and pass
`mizan pack validate` with zero errors.
