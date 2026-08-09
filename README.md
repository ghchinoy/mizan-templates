# mizan-templates

**Canonical, community-maintained template packs for [Mizan](https://github.com/ghchinoy/mizan).**

Mizan is a Go tool over the Vertex AI Gen AI Evaluation Service for creating,
managing, sharing, and running Gemini "LLM-as-a-Judge" metric templates across all
modalities (text, image, audio, video, music). This repository is the **shared
contribution channel** for those templates: it holds the canonical **template
packs** that Mizan users import from and contribute to.

> This repo contains **data + CI only — no application code.** The Mizan CLI/GUI
> and its Go source live in [`ghchinoy/mizan`](https://github.com/ghchinoy/mizan).
> The pack file format, the `Store`/`Codec`/`SyncBackend` seam, and the full design
> rationale are documented in that repo under `docs/collaboration-design.md`.

## Why a separate repo?

Template contribution and review are decoupled from the Mizan **source** repo:

- **Governance / access** — a collaborator can be granted PR rights here without
  any access to Mizan's application source.
- **Independent cadence** — packs version and release on their own schedule.

The trade-off (made explicit): this repo has no Go toolchain, so the `validate-packs`
CI must **obtain** a pinned `mizan` binary rather than build one — see
[`.github/workflows/validate-packs.yml`](.github/workflows/validate-packs.yml) and
the "CI / validation gate" section below.

## Repository layout

```
mizan-templates/
  README.md                      # this file
  docs/pack-format.md            # pack + template file-format reference
  packs/                         # <-- canonical shared packs (one subdir per pack)
    <pack-name>/
      mizan-pack.yaml            # pack manifest; metadata.name is the namespace
      templates/
        <slug>.yaml              # one MetricTemplate per file
      examples/                  # OPTIONAL: sample inputs / gs:// refs (docs only)
      rubrics/                   # OPTIONAL: shared rubric groups (P2+)
      README.md                  # pack description
  .github/workflows/validate-packs.yml   # PR gate, path-filtered to packs/**
```

- **One file per template** so independent contributions never merge-conflict.
- Every template's `metadata.id` is namespaced `"<pack-name>/<slug>"`, so two packs
  can each define e.g. a `helpfulness` metric without colliding on import.

A worked example pack lives at [`packs/google-brand/`](packs/google-brand/).

## Consuming packs

With the Mizan CLI installed, import from this repo (it is the **default source**):

```bash
# imports every pack under packs/ (this repo is the built-in default)
mizan registry import

# equivalently, explicitly:
mizan registry import github.com/ghchinoy/mizan-templates

# narrow to one namespace, or preview without writing:
mizan registry import --namespace google-brand
mizan registry import --dry-run
```

Imported templates land in your local working copy (SQLite) and are runnable:

```bash
mizan eval run --metric google-brand/video-brand-alignment \
  --field response=gs://your-bucket/ad.mp4 \
  --field brand_guideline="Our brand voice is warm, concise, never salesy."
```

> Multimodal assets must be `gs://` URIs — the native Eval Service does not accept
> inline bytes. See the Mizan docs.

## Contributing a template (PR workflow)

1. **Author locally**, then export into a pack directory in your clone of this repo:
   ```bash
   mizan registry export --id google-brand/my-metric --out packs/google-brand
   # writes packs/google-brand/templates/my-metric.yaml
   ```
   To start a brand-new pack: `mizan pack init packs/<name> --name <namespace>`.

2. **Validate locally** before pushing (creds-free, fast):
   ```bash
   mizan pack validate .
   ```

3. **Open a PR** against this repo:
   ```bash
   git checkout -b add-google-brand-my-metric
   git add packs/google-brand/ && git commit -m "google-brand: add my-metric"
   git push && gh pr create
   ```

4. **CI validates** your pack (see below). A maintainer reviews the prompt diff —
   YAML block scalars keep multi-line prompts readable in review — and merges.

5. After merge, any consumer picks it up with `mizan registry import`.

See [`docs/pack-format.md`](docs/pack-format.md) for the authoritative file format.

## CI / validation gate

The `validate-packs` workflow runs on PRs that touch `packs/**` and runs
`mizan pack validate .` (structural → identity → semantic → placeholder → lint;
all creds-free, no eval API calls). Because this repo has no Go source, the
workflow **obtains a version-pinned `mizan` validator** via `go install`.

> ⚠️ **The workflow is not yet installed at `.github/workflows/`.** The token used
> to scaffold this repo lacked GitHub `workflow` scope, so it could not create
> files under `.github/workflows/**`. The complete workflow stub therefore lives at
> [`ci/validate-packs.yml`](ci/validate-packs.yml); a repo admin with workflow
> scope must move it into place. See [`ci/README.md`](ci/README.md).

**Three one-time setup steps are required before the gate is active:**

1. **Install the workflow** — move `ci/validate-packs.yml` to
   `.github/workflows/validate-packs.yml` (see [`ci/README.md`](ci/README.md)).
2. **Secret `MIZAN_RO_TOKEN`** — a read-scoped token (fine-grained PAT or GitHub
   App installation token) with *contents:read* on `ghchinoy/mizan`, so CI can
   `go install` the (private) validator. Add under *Settings → Secrets and
   variables → Actions*.
3. **Pin `MIZAN_VERSION`** in the workflow to a tagged `mizan` release. Until a
   release is tagged, the workflow uses a `main` pseudo-version (or is left
   disabled); the pin is always explicit in the file, never hidden.

The alternatives (prebuilt release binary; cross-repo reusable workflow) and the
rationale for the pinned `go install` approach are documented in
`ghchinoy/mizan` → `docs/collaboration-design.md` §3.7.

## License

Individual packs declare their own `license` in `mizan-pack.yaml` and per template
`metadata.license`. Add a repository `LICENSE` file to set the default for
contributions.
