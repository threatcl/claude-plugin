---
description: Generate CI/CD scaffolding — Threatcl HCL validation and policy evaluation, or PR threat-model drift review via threatcl/drift-action
argument-hint: <github-actions|gitlab-ci|pre-commit|drift>
---

You are scaffolding CI integration for Threatcl. The flavor is `$ARGUMENTS` — one of: `github-actions`, `gitlab-ci`, `pre-commit`, `drift`. If the argument is empty, ask the user which flavor and stop.

The first three flavors scaffold **Threatcl Cloud** validation and policy evaluation. The `drift` flavor scaffolds [`threatcl/drift-action`](https://github.com/threatcl/drift-action), which reviews each PR for threat-model *drift* — it needs no Threatcl Cloud account and uses a different auth model. They're complementary: a repo can run both.

Before generating anything, scan the repo for existing CI config (`.github/workflows/`, `.gitlab-ci.yml`, `.pre-commit-config.yaml`, `.github/workflows/threat-drift.yml`, `.threatcl-ci.hcl`) and prefer to extend those rather than overwrite. If a Threatcl-related file already exists, show the user the diff before writing.

## Auth in CI

For the Threatcl Cloud flavors (`github-actions`, `gitlab-ci`, `pre-commit`), CI authenticates via API tokens, not the OAuth/device flow. Those scaffolds must:

- Reference a secret named `THREATCL_API_TOKEN` (the user creates it in the Threatcl Cloud UI and stores it as a CI secret)
- Set `THREATCL_API_URL` to the org's API endpoint (currently `https://beta-api.threatcl.com`; document this)
- Optionally set `THREATCL_CLOUD_ORG` to pin the org

The `drift` flavor does **not** use `THREATCL_API_TOKEN`. drift-action is self-hosted: your code and threat model go only to the LLM provider you configure, under your own key, never to Threatcl. Its secret is `ANTHROPIC_API_KEY` (or `OPENAI_API_KEY`).

## What each flavor produces

### github-actions

Write `.github/workflows/threatcl.yml` with two jobs:

1. **validate** (on `pull_request`) — checks out the repo, installs the threatcl CLI, runs `threatcl cloud validate` on every `*.hcl` file changed in the PR. Fails the job on any validation error.
2. **policy-evaluate** (on `push` to `main` or `pull_request` if the user prefers) — for each pushed model, run `threatcl cloud policy evaluate -model-id <id> -fail-on-error`. The model IDs come from the backend block in each HCL file — parse it. The backend block carries `organization` and `threatmodel` only; never emit a `segment` attribute, which was removed in spec 0.6.0 and is now an "Unsupported argument" parse error.

Pin the threatcl CLI install method (recommend `go install github.com/threatcl/threatcl/cmd/threatcl@latest` or a release-asset download). Use `actions/checkout@v7` and a Go setup action.

### gitlab-ci

Write `.gitlab-ci.yml` (or extend it) with `validate-threatcl` and `evaluate-threatcl-policies` jobs. Same auth model. Use `rules:` to scope validation to MRs and policy evaluation to default-branch pushes.

### pre-commit

Write `.pre-commit-config.yaml` (or extend it) with a local hook that runs `threatcl validate` on changed `*.hcl` files. Pre-commit is local-only — don't run cloud commands here, since contributors may not be authenticated. Note this in a comment.

### drift

Scaffolds [`threatcl/drift-action`](https://github.com/threatcl/drift-action) — the CI counterpart of this plugin's `/threat-drift` command. On each PR it checks the diff against the repo's threat model and posts a single sticky comment with evidence-backed findings across the same six drift categories.

**First, confirm the repo has a discoverable threat model.** The action discovers `*.tm.hcl` at the repo root, or `*.hcl` under `threatmodels/` or `threatmodel/`. A bare `*.hcl` at the root is deliberately *not* discovered — that glob would sweep up the action's own config file. If the repo's model doesn't match one of those, either suggest renaming it to `*.tm.hcl` or set `model_paths` in the config file (below). If there's no model at all, say so and suggest `/threat-hcl-new` first — there's nothing to drift from.

Write `.github/workflows/threat-drift.yml`:

```yaml
name: threat-drift

on:
  pull_request:

# One review in flight per PR. A push supersedes the run for the previous
# commit rather than racing it for the sticky comment.
concurrency:
  group: threat-drift-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write
  checks: write

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: threatcl/drift-action@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Four things in that block are load-bearing. Don't drop them, and explain them if the user asks:

- **`pull_request`, never `pull_request_target`.** Non-negotiable. `pull_request_target` would expose the API key and a write-scoped token to code from a fork. Under `pull_request`, fork PRs simply run without secrets and the comment says the change couldn't be assessed — rather than posting a shallow result that looks like a review.
- **The `concurrency` block.** The review takes minutes of inference. Without PR-keyed concurrency and `cancel-in-progress: true`, two rapid pushes race to update the same sticky comment and the last writer wins — possibly with findings from the older commit.
- **A pinned ref, never `@main`.** See below.
- **The `checkout` step.** The diff comes from the GitHub compare API, but the action reads whole files out of the workspace for context — without a checkout it reviews against an empty tree.

Minimal permissions are exactly the three above: `contents: read` to check out, `pull-requests: write` for the comment, `checks: write` for the check run.

#### Pinning

Default to `threatcl/drift-action@v1` — the latest v1.x release, which is what most repositories want. Tell the user it's a moving alias: it's force-moved onto each new release, so the engine reviewing their PRs can change without them editing anything. Since this job holds an LLM API key and a token that can write to pull requests, offer the immutable alternative:

```yaml
      - uses: threatcl/drift-action@<commit-sha> # v1.0.0
```

Dependabot bumps the SHA and reads the trailing comment to track the version. drift-action's own drift workflow pins the SHA for exactly this reason. If the user wants this, look up the current release SHA rather than guessing one.

#### `.threatcl-ci.hcl`

Every setting is optional — without the file, the action discovers the single model and uses its defaults. **Only write `.threatcl-ci.hcl` if the user needs a non-default**, most commonly because the repo has more than one discoverable model (in which case the action refuses to guess and `model_paths` is required).

The full surface, with defaults:

```hcl
# Which model to assess. Required only when the repo has more than one.
model_paths = ["threatmodels/payments.hcl"]

# Restrict the drift categories assessed. Omit to run all six. Valid values:
# stale_assertion, phantom_control, unmodeled_surface, dfd_drift,
# dependency_drift, unclassified_data
categories = ["phantom_control", "stale_assertion", "dependency_drift"]

# Paths that must always be reviewed, even when a large diff is narrowed.
# A trailing slash matches by prefix; otherwise matched with path.Match, and
# a bare filename matches wherever it sits in the tree.
trigger_paths = ["src/payments/", "cmd/*.go"]

fail_mode = "never"   # never (default) | on-action-required

llm {
  provider    = "anthropic"   # anthropic (default) | openai
  model       = "claude-opus-5"
  effort      = "high"        # low | medium | high | xhigh | max
  max_tokens  = 32000         # bounds output INCLUDING thinking
  api_key_env = "ANTHROPIC_API_KEY"
}

limits {
  max_diff_files    = 200      # over this, no review runs at all
  max_patch_bytes   = 400000
  max_context_bytes = 200000
  narrow_above      = 50       # above this, narrow to security-relevant paths
}
```

Notes worth passing on:

- An unknown category, fail mode, effort level or provider is a **hard error**, not a silent default — a typo can't quietly disable a drift check.
- `model` and `api_key_env` follow from `provider`, so `llm { provider = "openai" }` on its own is usually enough. `anthropic` defaults to `claude-opus-5`, `openai` to `gpt-5.6-sol`. Both key inputs are forwarded to the container and the engine reads only the one its provider names — so once both keys are wired up in the workflow, switching provider is a config-file change, not a workflow change:
  ```yaml
        - uses: threatcl/drift-action@v1
          with:
            anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
            openai-api-key: ${{ secrets.OPENAI_API_KEY }}
  ```
  Only wire up the second key if the user actually wants the option — an unset input arrives as an empty string, which counts as no key.
- `fail_mode = "never"` is the default: the action comments but doesn't fail the build. Suggest `on-action-required` only once the user has seen a few real reviews.

#### Trying it first

Recommend a trial run before letting it comment:

```yaml
      - uses: threatcl/drift-action@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          dry-run: true
```

`dry-run` renders the full review into the job log without posting anything. A non-boolean value is a hard error rather than a silent `false`, so a typo can't post a comment the user thought they'd suppressed.

## Output

Write the file(s) directly into the repo (after showing the diff if one exists). Then print:

- The path(s) written
- The exact secrets the user must add to CI
  - Cloud flavors: `THREATCL_API_TOKEN`, plus URL and org if not defaulting
  - `drift`: `ANTHROPIC_API_KEY` (or `OPENAI_API_KEY`), added as a repository secret under Settings → Secrets and variables → Actions
- A one-line way to check the result locally
  - Cloud flavors: e.g. `THREATCL_API_TOKEN=… threatcl cloud validate ./threatmodels/*.hcl`
  - `drift`: run `/threat-drift` in this repo to get the same review locally, before the first PR lands

For the cloud flavors, if `-fail-on-error` and `-fail-on-warning` are both configurable for the user's strictness preference, ask them which they want before generating, and bake the choice into the workflow.

For `drift`, mention that the action's outputs (`findings-count`, `action-required-count`, `verdict`, `report-path`) are available to later steps if they want to gate on them — and that `verdict` distinguishes `clean` (assessed, no drift) from `unassessed` (nothing was actually reviewed), so a gate can't mistake silence for safety.
