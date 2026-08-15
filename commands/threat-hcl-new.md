---
description: Scaffold a new Threatcl HCL threat model file with the cloud backend block pre-populated
argument-hint: <model-name> [optional one-line description]
---

You are creating a new Threatcl HCL threat model file. The argument string `$ARGUMENTS` is the model name (and optionally a short description after the first word group). If it's empty, ask the user for a name and stop.

## 1. Resolve the org slug

Run `threatcl cloud whoami`. Capture the **org slug** from the output — it goes into the `backend` block. If multiple orgs are listed, ask which one to scaffold for.

If `whoami` fails with an auth error, tell the user to run `threatcl cloud login` and stop. Don't try to write the file with a placeholder slug — `threatcl cloud push` will fail anyway.

## 2. Pick a filename

Slugify the model name to kebab-case and suffix with `.tm.hcl` (e.g. `Payment Service` → `payment-service.tm.hcl`). Write to the current working directory unless the user already has a `threatmodels/` or `models/` directory — in which case write there.

The `.tm.hcl` suffix is threatcl's convention and it matters for tooling: [`threatcl/drift-action`](https://github.com/threatcl/drift-action) discovers `*.tm.hcl` at the repo root (a bare root-level `*.hcl` is deliberately skipped so its own `.threatcl-ci.hcl` config is never mistaken for a model), and editors scope the `threatcl lsp` language server on the same suffix to avoid colliding with Terraform. The `threatcl` CLI itself accepts either.

If the chosen path already exists, stop and ask the user before overwriting.

## 3. Write the file

Use this template, filling in the org slug and model name:

```hcl
spec_version = "0.8.0"

backend "threatcl-cloud" {
  organization = "<org-slug>"
  # threatmodel slug is added automatically on first `threatcl cloud push`
}

threatmodel "<Model Name>" {
  description = "<one-sentence description, from $ARGUMENTS or a sensible default>"
  author      = "<git config user.name, falling back to '@team'>"

  attributes {
    new_initiative  = "true"
    internet_facing = "false"
    initiative_size = "Medium"
  }

  # Add information_asset blocks for the data this system handles, e.g.
  # information_asset "user records" {
  #   description                = "Names, emails, hashed passwords"
  #   information_classification = "Confidential"
  # }

  # Add usecase blocks describing what the system is supposed to do.
  # usecase {
  #   description = "Customers sign in to view their account"
  # }

  # Sample threat — replace or expand. Reference library items with `ref`
  # when one applies (search with `threatcl cloud search -type threats`).
  threat "Example threat — replace me" {
    description = "Describe the threat in one or two sentences"
    impacts     = ["Confidentiality"]
    stride      = ["Info Disclosure"]

    control "Example control — replace me" {
      description    = "How this threat is mitigated"
      implemented    = false
      risk_reduction = 50
    }
  }
}
```

Resolve `git config user.name` if available; otherwise leave `@team` as the author placeholder.

Names must be unique: as of spec 0.7.0, two `threat` blocks with the same name in one `threatmodel`, or two `control` blocks with the same name in one `threat`, are parse errors — not warnings. Keep that in mind when you later append to this file.

## 4. Report and suggest next steps

Print:

- The file path written
- The next two commands the user should run, in order:
  ```
  threatcl cloud validate <file>
  threatcl cloud push <file>
  ```
- A one-liner suggesting they run `/threat-for-code` against the relevant code paths to seed the model with concrete threats from the codebase.
- A one-liner suggesting `/threat-ci drift` to wire up automated threat-model drift review on every pull request, once the model has real content in it.

Don't run `validate` or `push` automatically — the user may want to flesh out the model first.
