# google-brand pack

Brand-compliance LLM-as-a-Judge templates for evaluating marketing assets against
brand guidelines.

- **Namespace:** `google-brand` (every template id begins `google-brand/`)
- **Maintainers:** `google-brand-team`
- **License:** Apache-2.0

## Templates

| id | kind | modalities | summary |
|---|---|---|---|
| `google-brand/video-brand-alignment` | pointwise | video, text | Scores how well a short video ad aligns with a supplied brand guideline (tone, visual identity, messaging). |

## Usage

```bash
mizan registry import --namespace google-brand
mizan eval run --metric google-brand/video-brand-alignment \
  --field response=gs://your-bucket/ad.mp4 \
  --field brand_guideline="Our brand voice is warm, concise, never salesy."
```

See `examples/` for sample inputs and the repository
[`docs/pack-format.md`](../../docs/pack-format.md) for the file format.
