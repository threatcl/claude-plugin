---
description: Generate CI/CD scaffolding that validates Threatcl HCL on PR and runs policy evaluation on merge
argument-hint: <github-actions|gitlab-ci|pre-commit>
---

You are scaffolding CI integration for Threatcl Cloud. The flavor is `$ARGUMENTS` — one of: `github-actions`, `gitlab-ci`, `pre-commit`. If the argument is empty, ask the user which flavor and stop.

Before generating anything, scan the repo for existing CI config (`.github/workflows/`, `.gitlab-ci.yml`, `.pre-commit-config.yaml`) and prefer to extend those rather than overwrite. If a Threatcl-related file already exists, show the user the diff before writing.

## Auth in CI

CI environments authenticate to Threatcl Cloud via API tokens, not the OAuth/device flow. The scaffolds you generate must:

- Reference a secret named `THREATCL_API_TOKEN` (the user creates it in the Threatcl Cloud UI and stores it as a CI secret)
- Set `THREATCL_API_URL` to the org's API endpoint (currently `https://beta-api.threatcl.com`; document this)
- Optionally set `THREATCL_CLOUD_ORG` to pin the org

## What each flavor produces

### github-actions

Write `.github/workflows/threatcl.yml` with two jobs:

1. **validate** (on `pull_request`) — checks out the repo, installs the threatcl CLI, runs `threatcl cloud validate` on every `*.hcl` file changed in the PR. Fails the job on any validation error.
2. **policy-evaluate** (on `push` to `main` or `pull_request` if the user prefers) — for each pushed model, run `threatcl cloud policy evaluate -model-id <id> -fail-on-error`. The model IDs come from the backend block in each HCL file — parse it.

Pin the threatcl CLI install method (recommend `go install github.com/threatcl/threatcl/cmd/threatcl@latest` or a release-asset download). Use `actions/checkout@v6` and a Go setup action.

### gitlab-ci

Write `.gitlab-ci.yml` (or extend it) with `validate-threatcl` and `evaluate-threatcl-policies` jobs. Same auth model. Use `rules:` to scope validation to MRs and policy evaluation to default-branch pushes.

### pre-commit

Write `.pre-commit-config.yaml` (or extend it) with a local hook that runs `threatcl validate` on changed `*.hcl` files. Pre-commit is local-only — don't run cloud commands here, since contributors may not be authenticated. Note this in a comment.

## Output

Write the file(s) directly into the repo (after showing the diff if one exists). Then print:

- The path written
- The exact secrets the user must add to CI (`THREATCL_API_TOKEN`, plus URL and org if not defaulting)
- A one-line test command the user can run locally to mirror what CI will do (e.g. `THREATCL_API_TOKEN=… threatcl cloud validate ./threatmodels/*.hcl`)

If `-fail-on-error` and `-fail-on-warning` are both configurable for the user's strictness preference, ask them which they want before generating, and bake the choice into the workflow.
