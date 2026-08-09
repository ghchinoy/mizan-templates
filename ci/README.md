# CI stub — `validate-packs`

`validate-packs.yml` in this directory is the **template-pack validation workflow**
for this repo. It belongs at **`.github/workflows/validate-packs.yml`**.

## Why it lives here (temporarily) instead of `.github/workflows/`

It was committed under `ci/` because the token used to scaffold this repo lacked
the GitHub **`workflow`** scope (fine-grained: *Workflows: write*), so it could not
create files under `.github/workflows/**` — GitHub rejects both `git push` and the
Contents API for workflow paths without that scope. The workflow content is
complete and correct; it just needs to be **moved into place by someone whose token
has workflow scope** (any repo admin acting through the GitHub UI or a
workflow-scoped token).

## Activation (one-time)

1. Move the file into place and commit (UI: "Add file → Create new file", paste the
   contents at path `.github/workflows/validate-packs.yml`; or with a
   workflow-scoped token: `git mv ci/validate-packs.yml .github/workflows/` and push).
2. Add the Actions secret **`MIZAN_RO_TOKEN`** — a read-scoped token
   (fine-grained PAT or GitHub App token) with *contents:read* on
   `ghchinoy/mizan`, so `go install` can fetch the (private) validator.
3. Pin **`MIZAN_VERSION`** in the workflow to a tagged `mizan` release once one
   exists (until then it resolves `main`).

Until steps 1–3 are done, there is **no active pack-validation gate** on PRs. This
is intentional and visible — see the repository `README.md` (CI section) and the
Mizan design doc `docs/collaboration-design.md` §3.7/§7 in `ghchinoy/mizan`.
