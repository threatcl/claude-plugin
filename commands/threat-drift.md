---
description: Detect drift between recent code changes and the documented Threatcl threat model
argument-hint: [diff-range]
---

You are checking whether recent code changes have drifted from the documented threat model. The argument `$ARGUMENTS` is a git ref range (e.g. `main...HEAD`, `HEAD~5..HEAD`, `v1.2.0...HEAD`). If empty, default to `main...HEAD` — the current branch vs main, which is the natural PR-review scope.

This is a **diagnostic** command, not a discovery command. You're checking the model's claims against current code reality, not suggesting new threats. (For "what threats apply to this code?" use `/threat-for-code` instead.)

## When the model is too vague to drift

Drift detection requires the threat model to make falsifiable claims about the code — things like "passwords are hashed with BCrypt in `auth.go`", "the API gateway authenticates via JWT", "no PII flows to third_party `Sentry`". Models full of generic threats like "an attacker exploits the API" can't drift because they don't assert anything checkable.

If, after reading the model in step 2, you find it's almost entirely vague language with no concrete code references, no DFD, no `information_asset` blocks, and no `third_party_dependency` blocks — say so directly to the user and stop. Don't manufacture low-confidence drift findings. Suggest they enrich the model first (a DFD via `data_flow_diagram_v2`, named information assets, specific control implementations) and re-run.

## 1. Resolve the diff

```bash
git diff $ARGUMENTS --stat   # overview
git diff $ARGUMENTS          # full diff for analysis
```

If the range is invalid (branch doesn't exist, no common ancestor), tell the user and stop.

If the diff is enormous (hundreds of files), narrow to security-relevant paths first: auth, API handlers, data access, crypto, network, dependency manifests (`go.mod`, `package.json`, `requirements.txt`, etc.), config files, and anything matching paths the threat model explicitly references.

## 2. Load the threat model

Find the local `.hcl` threat model file — usually one or two at the repo root or under `threatmodels/`. Read it fully. Note:

- Each `threat` block's description and any file/line references in the prose
- Each `control` block's `implemented` flag and its description
- `information_asset` blocks (what data the model says exists)
- `third_party_dependency` blocks (what vendors the model says are involved)
- `data_flow_diagram_v2` blocks (processes, data_stores, external_elements, flows, trust_zones)

If multiple HCL files exist, ask the user which model to drift-check against. Don't guess.

If no HCL file exists in the repo at all, tell the user — there's nothing to drift from. Suggest `/threat-hcl-new` to scaffold one.

## 3. Run drift checks

For each category below, walk the diff and compare against the model. **Cite specific file:line evidence for every finding.** A drift finding without a code reference is a guess.

### Stale assertions

For each `threat` whose description references concrete code or behavior ("uses BCrypt", "validates JWT in middleware", "rate-limits to 100 rps"), check whether that assertion still holds in the changed code. If the diff shows the implementation moved away from what the threat describes, flag it.

### Phantom controls

For each `control` with `implemented = true`, look for evidence the implementing code is still present and functioning. If the diff removes or substantially rewrites the code that backed an "implemented" control, flag it — the model is now lying.

### New unmodeled surface

Look in the diff for:
- New HTTP/gRPC endpoints, routes, handlers
- New external API calls (HTTP clients, SDK invocations)
- New crypto operations
- New file I/O at security-sensitive paths
- New deserialization or templating
- New auth/session paths

For each, check if any threat or DFD element in the model represents it. If not, that's new attack surface the model doesn't see.

### DFD drift

If the model has a `data_flow_diagram_v2`:
- New external dependencies in code → is there a corresponding `external_element`?
- New data flows in code (HTTP calls, DB queries) → is there a corresponding `flow`?
- Removed code paths → are any `process`/`data_store`/`flow` blocks now orphaned?
- Trust boundary crossings introduced by the diff → does the DFD's `trust_zone` topology still hold?

### `third_party_dependency` drift

Diff the dependency manifests (`go.mod`, `package.json`, `requirements.txt`, `Gemfile`, etc.):
- Newly added third-party deps that should be `third_party_dependency` blocks but aren't
- Removed deps that still appear as `third_party_dependency` blocks (orphan)
- Major version bumps of deps the model lists as `uptime_dependency = "operational"` (worth flagging — operational deps that just became a different vendor product are noteworthy)

### Unclassified data

New struct/class fields, DB columns, or API response fields in the diff that look like personal data, secrets, or sensitive information. Compare against the model's `information_asset` blocks — if there's no asset that covers the new data, flag it.

## 4. Output

Format:

```
# Drift report: <diff range> vs <model name>

## Summary
<one sentence: drift severity, e.g. "3 stale assertions, 1 phantom control, 4 new surfaces. The phantom control is the highest priority.">

## Stale assertions
- Threat "<name>" — said: <quoted description excerpt>
  Code now: <what the diff shows> at <file>:<line>
  Suggested fix: <e.g. "update threat description, or revisit if the threat is still valid">

## Phantom controls
- Control "<name>" on threat "<threat name>" — implementing code at <file>:<line> was <removed | rewritten | moved>
  Suggested fix: <e.g. "set implemented = false, or update the description to point at the new implementation">

## New unmodeled surface
- <file>:<line> — <what was added, e.g. "new POST /api/refund handler with no auth check"> — not represented in the model
  Suggested fix: <e.g. "add a threat block for unauthenticated refund issuance, or extend an existing API auth threat">

## DFD drift
- <issue>
  Suggested fix: <e.g. "add `flow \"https\" { from = \"web\" to = \"new-billing-service\" }\"`">

## Third-party dependency drift
- <issue>
  Suggested fix: <e.g. "add a third_party_dependency block for X, or remove the obsolete block for Y">

## Unclassified data
- <issue>
  Suggested fix: <e.g. "add `information_asset \"customer_phone\"` with classification = Confidential">

## Suggested next actions
1. <ordered, concrete edits to the HCL>
…
```

Omit any section with zero findings — don't pad the report. If the entire report is empty, say so plainly: "No drift detected between `<range>` and `<model name>`. The model is consistent with the changes."

## Don't write to the HCL

The user reviews the report and decides what to act on. Don't auto-edit the threat model file unless the user explicitly asks ("apply the suggested fixes" or similar). If they do ask, propose the diff first, then apply only after confirmation.
