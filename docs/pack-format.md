# Mizan pack & template format reference

This is a consumer-facing projection of the authoritative design in
[`ghchinoy/mizan`](https://github.com/ghchinoy/mizan) → `docs/collaboration-design.md`
(§3.2 format, §3.3 layout, §3.4 versioning, §3.5 validation). Where this file and
the Mizan design/JSON-Schema disagree, **the Mizan repo is authoritative** — the
`mizan pack validate` binary (and its shipped JSON Schema) are the single source of
structural truth.

- Format version: `apiVersion: mizan.dev/v1alpha1`
- File encoding: **YAML** (chosen so multi-line prompts diff line-by-line in PR review)

---

## 1. Pack directory

A **pack** is a directory under `packs/`:

```
packs/<pack-name>/
  mizan-pack.yaml        # REQUIRED — pack manifest
  templates/*.yaml       # REQUIRED — one MetricTemplate per file
  examples/*.yaml        # OPTIONAL — sample inputs / gs:// refs (documentation only)
  rubrics/*.yaml         # OPTIONAL — shared rubric groups (P2+; not required)
  README.md              # OPTIONAL — human description of the pack
```

- `<pack-name>` (the directory name) **is the namespace** and must equal
  `metadata.name` in `mizan-pack.yaml`.
- Templates are discovered by globbing `templates/*.yaml`. The manifest does **not**
  enumerate templates (that would re-introduce a merge-conflict surface).

### `mizan-pack.yaml` (manifest)

```yaml
apiVersion: mizan.dev/v1alpha1
kind: Pack
metadata:
  name: google-brand            # the namespace; must match the directory name
  version: 1.0.0                # pack-level semver (human release tag)
  description: Brand-compliance judge templates for marketing assets.
  maintainers: [google-brand-team]
  license: Apache-2.0
spec:
  requiresApiVersion: mizan.dev/v1alpha1   # min format version to consume this pack
```

---

## 2. Template file (`kind: MetricTemplate`)

One template per file in `templates/`. Filename convention: `<slug>.yaml` where the
slug matches the id suffix.

```yaml
apiVersion: mizan.dev/v1alpha1
kind: MetricTemplate

metadata:
  id: google-brand/video-brand-alignment   # STABLE id: "<namespace>/<slug>"
  name: Video Brand Alignment
  description: >
    Scores whether a short video ad aligns with a supplied brand guideline.
  version: 1.0.0                  # semver; bump on any spec change
  authors:
    - name: Jane Doe
      email: jane@example.com
  maintainers: [google-brand-team]
  license: Apache-2.0
  tags: [advertising, brand-safety, video]
  # contentHash is COMPUTED by the tool (not authored); present in exports.

spec:
  kind: pointwise                # pointwise | pairwise | rubric | custom_schema
  modalities: [video, text]      # asset types this template accepts

  inputs:                        # declared placeholders + modality
    - name: response             # the asset under evaluation
      modality: video
      required: true
    - name: brand_guideline      # reference text
      modality: text
      required: true

  metricPromptTemplate: |        # {{placeholder}} substitution (double-brace)
    You are a brand compliance rater. Given a brand guideline and a video ad,
    rate how well the ad aligns with the guideline on a 1-5 scale.

    Brand guideline:
    {{brand_guideline}}

    Evaluate the video: {{response}}

  systemInstruction: |
    Be strict. Penalize off-brand tone even when production quality is high.

  autorater:
    # Publisher-relative model id ONLY (e.g. "gemini-2.5-pro" or
    # "publishers/google/models/gemini-2.5-pro"). Do NOT embed a project/location —
    # packs are portable. Mizan expands this to the full resource name the Eval
    # Service requires at run time, using the consumer's configured project/location.
    model: gemini-2.5-pro
    samplingCount: 4             # 1-32
    flipEnabled: false           # pairwise-only; ignored otherwise
```

### Kind-specific fields

| `spec.kind`     | Additional required fields |
|-----------------|----------------------------|
| `pointwise`     | none (forbids the pairwise/rubric/custom fields below) |
| `pairwise`      | `candidateFieldName`, `baselineFieldName` (both also in `inputs`) |
| `rubric`        | `rubricGroups: { <group>: [ "<criterion>", ... ] }` (non-empty) |
| `custom_schema` | `responseSchema: { ...JSON Schema... }` (routes to the genai fallback) |

---

## 3. Placeholders

- Author placeholders as **double-brace `{{name}}`** in `metricPromptTemplate` and
  `systemInstruction`. (The API accepts single-brace too, but Mizan standardizes on
  and validates double-brace.)
- Every `{{name}}` used **must** be declared in `spec.inputs`; every `required`
  input **must** be referenced; each input's `modality` must be listed in
  `spec.modalities`.

---

## 4. Multimodal assets

Non-text assets are supplied at eval time as **`gs://` URIs** — the native Eval
Service does **not** accept inline bytes. `examples/*.yaml` may reference small
sample `gs://` inputs for documentation, but the pack itself carries no media.

---

## 5. Versioning & integrity

| Axis | Field | Bumped by |
|---|---|---|
| Format version | `apiVersion` | Mizan releases |
| Pack version | `mizan-pack.yaml` `metadata.version` | maintainer |
| Template version | template `metadata.version` (semver) | contributor (drives import conflict resolution) |

`contentHash` is computed by Mizan over the canonicalized spec + identity (never
hand-authored) and is used for drift detection and no-op import short-circuiting.
Git history is the real provenance ledger; `authors`/`maintainers` are the asserted
attribution.

---

## 6. Validation (what CI enforces)

`mizan pack validate .` runs, per template, failing the PR on any error:

1. **Structural** — validates against the shipped `MetricTemplate` JSON Schema.
2. **Identity** — `id == "<pack-name>/<slug>"`, slug `[a-z0-9-]+`, unique in pack;
   `version` parses as semver.
3. **Semantic** — kind-specific required fields (table in §2).
4. **Placeholder consistency** — the `{{...}}` ↔ `spec.inputs` rules in §3.
5. **Lint (warnings)** — missing `description`/`license`; `samplingCount` outside
   1–32; no `autorater.model`.

Steps 1–5 are creds-free (no eval API calls). An optional `--dry-run` performs one
live trial eval and is **not** run in CI.
