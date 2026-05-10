---
name: threatcl-cloud
description: >
  Threat modeling with Threatcl Cloud. Use when the user asks about threat
  models, security threats, controls, STRIDE analysis, HCL threat model files,
  or wants to interact with their Threatcl Cloud organization. Requires the
  `threatcl` CLI to be installed.
---

# Threatcl Cloud — Agent Skill

You are helping a security or software engineer work with threat models using Threatcl Cloud and the `threatcl` CLI.

## How this plugin is wired up

This skill ships as part of the `threatcl-cloud` Claude Code plugin. Two channels connect to Threatcl Cloud:

- **MCP server** (`threatcl`) — bundled with the plugin. Exposes tools like `list_threat_models`, `get_threat_model`, `search`, `list_library_items` and more. Authenticated automatically via Claude Code's OAuth flow on first use. Prefer MCP tools for **read** operations: listing, viewing, searching, library browsing, usage analytics.
- **`threatcl` CLI** — installed locally by the user. Required for **write** operations and local-file work: `validate`, `push`, `library import`, `policy create/update`, generating DFDs, exporting to OTM, etc. Authenticated separately via `threatcl cloud login` (device flow).

When both work for the same task, prefer MCP — fewer round-trips, no CLI auth dependency. Fall back to the CLI when MCP doesn't expose the operation (anything that writes or touches local files).

## Prerequisites

Before running any `threatcl cloud` command in a session, confirm the CLI is available and authenticated:

1. `threatcl version` — confirm the CLI is installed
2. `threatcl cloud whoami` — confirm authentication and note the org slug

**If `whoami` fails with a connection error** (not an auth error), the CLI is probably pointed at the wrong endpoint. Today, `threatcl cloud` requires:
```bash
export THREATCL_API_URL=https://beta-api.threatcl.com
```
Set it in-session, and suggest the user persist it in their shell profile.

**If `whoami` fails with an auth error**, tell the user to run `threatcl cloud login` and complete the browser flow.

If the CLI isn't installed at all:
- macOS/Linux: `brew install threatcl`
- Go: `go install github.com/threatcl/threatcl/cmd/threatcl@latest`
- Releases: https://github.com/threatcl/threatcl/releases

For deeper reference docs: https://threatcl.dev/cloud/overview/

Every `threatcl cloud` subcommand supports `-h` — use it when you're unsure about flags rather than guessing.

## Core Workflows

### Listing and Viewing Threat Models

```bash
# List all threat models in the organization
threatcl cloud threatmodels

# View details of a specific model (use slug from the list)
threatcl cloud threatmodel -model-id <slug>

# Download a threat model's HCL file
threatcl cloud threatmodel -model-id <slug> -download=<filename>.hcl

# View version history
threatcl cloud threatmodel versions -model-id <slug>

# Download a specific previous version
threatcl cloud threatmodel versions -model-id <slug> -version <ver> -download <file>.hcl

# View a local HCL file enriched with cloud control library data
# (any `ref`-linked threats/controls have their cloud definitions inlined,
# while local overrides like description/risk_reduction are preserved)
threatcl cloud view <file>.hcl

# Output raw markdown instead of formatted terminal output (good for piping)
threatcl cloud view -raw <file>.hcl

# Skip fetching recommended controls linked from threat-library refs
threatcl cloud view -ignore-linked-controls <file>.hcl

# View a cloud-hosted threat model directly by ID or slug — no local file needed
threatcl cloud view -model-id <slug>

# Same, but override the organization
threatcl cloud view -model-id <slug> -org-id <orgId>
```

`threatcl cloud view` is the preferred way to inspect a model when library refs are in use — `threatcl view` (the local-only command) shows the raw `ref = "..."` lines without resolving them, whereas `threatcl cloud view` renders the fully-enriched markdown.

### Searching Threats and Controls within existing Threat Models

```bash
# Search threats by impact
threatcl cloud search -impacts "Confidentiality"

# Search by STRIDE category
threatcl cloud search -stride "Spoofing,Tampering"

# Find threats without controls (security gaps)
threatcl cloud search -has-controls=false

# Search controls
threatcl cloud search -type controls

# Find implemented controls only
threatcl cloud search -type controls -implemented=true

# Scope to a specific threat model
threatcl cloud search -threatmodel-id "<uuid>"

# Combine filters
threatcl cloud search -impacts "Confidentiality" -stride "Info Disclosure" -has-controls=true
```

### Creating and Pushing Threat Models

Every HCL file that syncs to Threatcl Cloud needs a `backend` block. The `organization` slug comes from `threatcl cloud whoami` — if the user doesn't know it, run that first.

```hcl
spec_version = "0.2.4"

backend "threatcl-cloud" {
  organization = "<org-slug>"
  threatmodel  = "<model-slug>"   # added automatically on first push
}

threatmodel "My Application" {
  description = "Description of the system"
  author      = "@team-name"

  threat "Example Threat" {
    description = "Description of the threat"
    impacts     = ["Confidentiality", "Integrity"]
    stride      = ["Spoofing"]

    control "Example Control" {
      description    = "How we mitigate this threat"
      implemented    = true
      risk_reduction = 75
    }
  }
}
```

```bash
# Validate the file against the cloud org (checks backend block, org membership)
threatcl cloud validate <file>.hcl

# Push to cloud (creates model on first push, uploads new version on subsequent)
threatcl cloud push <file>.hcl
```

On first push, `threatcl cloud push` will:
1. Create the threat model in the organization
2. Add the `threatmodel` attribute to the backend block in the local file
3. Upload the HCL as the first version

On subsequent pushes, it will only push if differences are detected.

### Working with Libraries

```bash
# Export the organization's threat/control library
threatcl cloud library export
threatcl cloud library export -type threats -o threats.hcl
threatcl cloud library export -type controls -tags "owasp" -o controls.hcl
threatcl cloud library export -include-drafts -o full-library.hcl

# Import a library file
threatcl cloud library import library.hcl
threatcl cloud library import -mode update library.hcl

# List or search for threats
threatcl cloud library threats
threatcl cloud library threats -search "term"

# List or search for controls
threatcl cloud library controls
threatcl cloud library controls -search "term"
```

Library items can be referenced in threat models using `ref`:

```hcl
threat "sqli" {
  ref = "T-SQLI"    # pulls full definition from cloud library
}

control "logging" {
  ref = "C-LOGGING"  # pulls full definition from cloud library
}
```

### Working with Policies

Policies are Rego (Open Policy Agent) rules that evaluate threat models against organizational standards — for example, "every internet-facing model must have at least one Spoofing control" or "no threat may be unmitigated if it impacts Confidentiality." Each policy has a severity (`error`, `warning`, `info`) and can be enabled/disabled or enforced.

```bash
# List all policies in the organization
threatcl cloud policies

# List only enabled policies
threatcl cloud policies -enabled-only

# View a specific policy (add -show-rego to include the source)
threatcl cloud policy -policy-id <uuid>
threatcl cloud policy -policy-id <uuid> -show-rego

# Validate a local .rego file against the cloud (no create)
threatcl cloud policy validate ./my-policy.rego

# Create a new policy from a local .rego file
threatcl cloud policy create \
  -name "Internet-facing must mitigate Spoofing" \
  -severity error \
  -rego-file ./policy.rego \
  -description "Every internet-facing model needs at least one Spoofing control" \
  -category "auth" \
  -tags "internet-facing,spoofing"

# Update an existing policy
threatcl cloud policy update -policy-id <uuid> -severity warning
threatcl cloud policy update -policy-id <uuid> -rego-file ./updated.rego
threatcl cloud policy update -policy-id <uuid> -enabled=false

# Delete a policy (use -force to skip confirmation)
threatcl cloud policy delete -policy-id <uuid>
```

#### Evaluating Policies Against a Threat Model

```bash
# Run all enabled policies against a model
threatcl cloud policy evaluate -model-id <uuid>

# CI/CD-friendly: exit non-zero on failures
threatcl cloud policy evaluate -model-id <uuid> -fail-on-error
threatcl cloud policy evaluate -model-id <uuid> -fail-on-warning

# List past evaluation runs for a model
threatcl cloud policy evaluations -model-id <uuid>
threatcl cloud policy evaluations -model-id <uuid> -limit 50

# View a specific past evaluation
threatcl cloud policy evaluation -model-id <uuid> -eval-id <eval-uuid>
```

When helping a user author a new Rego policy, always run `threatcl cloud policy validate <file>.rego` before creating it — this catches syntax errors and schema mismatches without creating a half-broken policy in the org.

### Local-Only Operations (No Cloud Required)

The `threatcl` CLI also works purely locally:

```bash
# Validate HCL syntax (no cloud connection needed)
# Note: this does NOT validate backend blocks or other threatcl cloud
# attributes — use `threatcl cloud validate` for that.
threatcl validate <file>.hcl

# List threat models in local files
threatcl list <file>.hcl

# View a threat model's content
threatcl view <file>.hcl

# Generate a new threat model interactively
threatcl generate interactive

# Generate boilerplate
threatcl generate boilerplate

# Export to JSON/OTM format
threatcl export <file>.hcl
threatcl export -format otm <file>.hcl

# Generate data flow diagram
threatcl dfd <file>.hcl

# Generate markdown dashboard
threatcl dashboard <file>.hcl -outdir ./docs
```

## HCL Threat Model Structure

When helping users write or edit HCL threat model files, follow this structure:

```hcl
spec_version = "0.2.4"

backend "threatcl-cloud" {
  organization = "<org-slug>"
  threatmodel  = "<model-slug>"
}

threatmodel "<Name>" {
  description = "System description"
  author      = "@author"

  attributes {
    new_initiative  = "false"
    internet_facing = "true"
    initiative_size = "Medium"
  }

  information_asset "<asset-name>" {
    description                = "What this data is"
    information_classification = "Confidential"  # Restricted | Confidential | Public
  }

  usecase {
    description = "This is a valid use-case for the system being modeled"
  }

  exclusion {
    description = "This is something that is not being assessed in this model"
  }

  third_party_dependency "Cloud Provider" {
    description = "What this 3rd party does"

    # Optional boolean attributes
    saas = "true"
    paying_customer = "false"
    open_source = "true"
    infrastructure = "true"

    # Required - should be none, degraded, hard, operational
    uptime_dependency = "operational"

    # Optional notes
    uptime_notes = "If this goes down, so does our solution"
  }

  threat "<threat-name>" {
    description          = "What could go wrong"
    impacts              = ["Confidentiality", "Integrity", "Availability"]
    stride               = ["Spoofing", "Tampering", "Repudiation",
                            "Info Disclosure", "Denial of Service",
                            "Elevation of Privilege"]
    information_asset_refs = ["<asset-name>"]

    # Or reference a library item:
    # ref = "T-LIBRARY-REF"

    control "<control-name>" {
      description    = "How we mitigate this"
      implemented    = true
      risk_reduction = 75  # 0-100

      # Or reference a library item:
      # ref = "C-LIBRARY-REF"
    }
  }
}
```

Within a threatmodel block we can also include an optional data flow diagram:

```hcl
data_flow_diagram_v2 "Diagram name" {
  # All blocks must have unique names
  # That means that a process, data_store, or external_element can't all
  # be named "foo"

  process "update data" {}

  # All these elements may include an optional trust_zone
  # Trust Zones are used to define trust boundaries

  process "update password" {
    trust_zone = "secure zone"
  }

  data_store "password db" {
    trust_zone = "secure zone"

    # data_store blocks can refer to an information_asset from the
    # threatmodel
    information_asset = "cred store"
  }

  external_element "user" {}

    # To connect any of the above elements, you use a flow block
    # Flow blocks can have the same name, but their from and to fields
    # must be unique

  flow "https" {
    from = "user"
    to = "update data"
  }

  flow "https" {
    from = "user"
    to = "update password"
  }

  flow "tcp" {
    from = "update password"
    to = "password db"
  }

  # You can also define Trust Zones at the data_flow_diagram level

  trust_zone "public zone" {

    # Within a trust_zone you can then include processes, data_stores
    # or external_elements

    # Make sure that either you omit the element's trust_zone, or that it
    # matches

    process "visit external site" {}

    external_element "OIDC Provider" {}

  }
}
```

### STRIDE Categories Reference

- **Spoofing** — Impersonating something or someone
- **Tampering** — Modifying data or code
- **Repudiation** — Denying having performed an action
- **Info Disclosure** — Exposing information to unauthorized parties
- **Denial of Service** — Making a system unavailable
- **Elevation of Privilege** — Gaining access beyond authorization

## Behavioral Guidelines

1. **Validate before pushing** — Always run `threatcl cloud validate` before `threatcl cloud push`. Catches org mismatches and HCL errors early.

2. **Use search to find gaps** — When reviewing a threat model, use `threatcl cloud search -has-controls=false` to find unmitigated threats.

3. **Reference library items** — When adding threats or controls, check the library first with `threatcl cloud library controls`, `threatcl cloud library threats` or to look for existing controls within threat models you can `threatcl cloud search -type controls`. Use `ref` attributes to link to library items rather than duplicating definitions.

4. **Explain STRIDE** — When helping users categorize threats, explain which STRIDE categories apply and why.

5. **Respect the backend block** — Never remove or modify the `backend` block unless the user explicitly asks. It links the local file to the cloud.

6. **Suggest data flow diagrams** — For complex systems, suggest adding a `data_flow_diagram_v2` block and generating a visual with `threatcl dfd`.

7. **Validate Rego before creating policies** — When authoring or modifying a policy, always run `threatcl cloud policy validate <file>.rego` before `threatcl cloud policy create` or `update -rego-file`. This catches Rego syntax errors and schema mismatches without leaving a broken policy in the org.

8. **Use `-fail-on-error` / `-fail-on-warning` in CI** — When wiring `threatcl cloud policy evaluate` into CI/CD, use these flags so that policy violations actually break the build. Without them the command always exits 0 regardless of result.

## Environment Variables (for CI/CD context)

If running in CI/CD, these env vars are available instead of interactive auth:

- `THREATCL_API_TOKEN` — API token (bypasses local token store)
- `THREATCL_CLOUD_ORG` — Default organization ID
- `THREATCL_API_URL` — API base URL override (currently required: `https://beta-api.threatcl.com`)
