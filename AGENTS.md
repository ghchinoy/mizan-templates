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
- **`EvalSet` CLI & Cross-Kind Gate Calibration**:
  - Execute `EvalSet` suites via `mizan eval run --set packs/<pack-name>/evalsets/<slug>.yaml` (there is no `mizan evalset` subcommand).
  - Phase-1 `mean` / `weighted-mean` aggregates raw numeric `Score` values without cross-kind normalization (`rubric`/`score` on `1..5`, `boul` mapping `PASS`/`FAIL` to `1.0`/`0.0`, while `choice` and `pairwise` have `Score == nil` and render as non-scalar verdict rows). When mixing `1..5` rubrics with `0/1` `boul` safety checks under `gate: true`, calibrate `aggregation.threshold` high enough (e.g., `2.2` in `packs/multimodal-judge/evalsets/multimodal-judge-core.yaml`) so a `boul` failure (`0.0`) always trips the gate.

## Validation & Local Sync

- **Creds-Free Validation (Required Before Every Commit/PR)**:
  ```bash
  # Using a local ../mizan checkout (preferred when developing alongside mizan):
  ../mizan/bin/mizan pack validate .

  # Or using an installed mizan CLI (v0.4.0+):
  mizan pack validate .
  ```
  Ensure `0 error(s), 0 warning(s)`.
- **Local Registry Sync**:
  ```bash
  mizan registry import ./packs/<pack-name> --strategy overwrite
  ```
  *(Note: the subcommand is `mizan registry import`, not `mizan pack import`.)*
- **CI Validator Pin & Fork Safety**: `.github/workflows/validate-packs.yml` installs a tagged `ghchinoy/mizan` release via `MIZAN_VERSION`. Bump `MIZAN_VERSION` deliberately only after a new `ghchinoy/mizan` tag is published, and keep `MIZAN_RO_TOKEN` optional (never `exit 1` when empty) so external fork `pull_request` runs (where GitHub withholds `secrets.*`) pass anonymously against public `ghchinoy/mizan`.
- **Secrets Hygiene**: Never commit `.env`, `.env.*`, or credentials.
