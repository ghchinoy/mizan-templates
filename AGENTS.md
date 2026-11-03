# Mizan-Templates Agent Guidelines

## Repository Scope & Separation

- **Data + CI Only (`ghchinoy/mizan-templates`)**: This repository holds canonical, community-maintained `MetricTemplate` and `EvalSet` packs under `packs/<pack-name>/`, format docs under `docs/`, and the `.github/workflows/validate-packs.yml` validation gate.
- **No Application or Go Source Code**: Never add Go files, CLI code, or validator tests here. Changes to `mizan` CLI commands, `ValidatePack` rules, JSON Schemas, or the evaluation engine belong in separate PRs against [`github.com/ghchinoy/mizan`](https://github.com/ghchinoy/mizan).
- **Split Mixed Work**: When developing both a new/updated pack and an upstream `mizan` engine/validator feature, ship the pack PR here and open a separate code/test PR in `ghchinoy/mizan`.

## Pack Layout & Authoring Conventions

- **Canonical Pack Structure**:
  ```text
  packs/<pack-name>/
    mizan-pack.yaml        # REQUIRED: kind: Pack; metadata.name == <pack-name>
    templates/<slug>.yaml  # REQUIRED: kind: MetricTemplate; metadata.id == "<pack-name>/<slug>"
    evalsets/<slug>.yaml   # OPTIONAL: kind: EvalSet; metadata.id == "<pack-name>/<slug>"
    examples/*             # OPTIONAL: sample .yaml inputs or compare-engines --dataset .jsonl cases
    README.md              # RECOMMENDED: pack overview, templates table, and usage notes
  ```
- **Use `examples/`, Not `fixtures/`**: Place sample inputs and `mizan eval compare-engines --dataset` JSONL files under `packs/<pack-name>/examples/` (see `packs/calibration/examples/`). If sample files reference illustrative `gs://` URIs that are not live public objects, state that explicitly in the pack's `README.md`. Never commit binary media payloads.
- **Format & Placeholders (`apiVersion: mizan.dev/v1alpha1`)**:
  - Author prompt placeholders with double braces (`{{field_name}}`). Every `{{field_name}}` in `metricPromptTemplate` and `systemInstruction` must match `spec.inputs`, and every input `modality` must appear in `spec.modalities`.
  - Keep `spec.autorater.model` publisher-relative (never embed `projects/<proj>/locations/<loc>/...`).
- **Durable Model Pins**:
  - Pin `spec.autorater.model` to `gemini-3.5-flash-lite` (default for `boul`, `choice`, and `score`) or `gemini-3.5-flash` (multimodal/pointwise/pairwise/rubric), unless benchmarks justify `gemini-3.8-flash`.
  - **Never** pin retired `gemini-2.x` models (`gemini-2.5-flash`, `gemini-2.5-pro`).
- **`boul` Polarity Convention**:
  - For condition detectors (toxicity, prompt injection, drift), define `passed: true` to mean *"the proposition holds / condition is detected"*.
  - For compliance checks, state `PASS (true)` and `FAIL (false)` conditions explicitly in `metricPromptTemplate`.

## Validation & Local Sync

- **Creds-Free Validation (Required Before Every Commit/PR)**:
  ```bash
  # Using a local ../mizan checkout (preferred when developing alongside mizan):
  ../mizan/bin/mizan pack validate .

  # Or using an installed mizan CLI (v0.3.0+):
  mizan pack validate .
  ```
  Ensure `0 error(s), 0 warning(s)`.
- **Local Registry Sync**:
  ```bash
  mizan registry import ./packs/<pack-name> --strategy overwrite
  ```
  *(Note: the subcommand is `mizan registry import`, not `mizan pack import`.)*
- **CI Validator Pin**: `.github/workflows/validate-packs.yml` installs a tagged `ghchinoy/mizan` release via `MIZAN_VERSION`. Bump `MIZAN_VERSION` deliberately only after a new `ghchinoy/mizan` tag is published.
- **Secrets Hygiene**: Never commit `.env`, `.env.*`, or credentials.
