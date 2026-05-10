---
description: Run a structured security review of a Threatcl Cloud threat model
argument-hint: <model-slug-or-id>
---

You are running a structured security review of the threat model identified by `$ARGUMENTS`. Gather data first, then synthesize. Be concrete — every finding must reference specific threats, controls, or policies you pulled, not generic security advice.

## 1. Load the model

Use the `get_threat_model` MCP tool (preferred) to fetch the model by slug or ID. Fall back to `threatcl cloud view -model-id $ARGUMENTS` if MCP is unavailable. Note: name, status, last-updated, threat count, control count.

## 2. Find unmitigated threats

Run the `search` MCP tool (or `threatcl cloud search`) scoped to this model with `has-controls=false`. List every threat with no mapped controls — these are the highest-priority gaps.

## 3. Tally STRIDE coverage

Across all threats in the model, count which STRIDE categories appear. Any category that's entirely absent is worth calling out, but only suggest adding threats in missing categories if they genuinely apply to the system being modeled.

## 4. Run policy evaluation

Run `threatcl cloud policy evaluate -model-id <id>`. Surface every error and warning verbatim — the policies are configured by the org and you shouldn't editorialize them away. If policy evaluation isn't available (no policies defined, MCP-only environment), say so and skip.

## 5. Cross-check the threat library

Use `list_library_items` (MCP) or `threatcl cloud library threats` to scan the org's threat library. Identify up to three library threats whose STRIDE/impacts/tags overlap with the model's domain but aren't yet referenced in it. Suggest each by `ref` ID and explain why it fits.

## 6. Output the report

Format:

```
# Review: <model name> (<slug>)

## Summary
<1–2 sentences: overall posture, biggest concern>

## Unmitigated threats (<count>)
- <threat name> — STRIDE: <category> — Impacts: <list>
…

## STRIDE coverage
- Spoofing: <covered count> threats
- Tampering: …
- (mark any absent categories explicitly)

## Policy evaluation
- Errors: <count> · Warnings: <count> · Info: <count>
- <severity>: <policy name> — <violation reason>
…

## Library suggestions
- T-<ref>: <name> — why this fits this model
…

## Recommended next actions
1. <concrete next step, e.g. "Add a control to threat 'X' mitigating Y">
…
```

If the model is in `approved` status but has unmitigated threats or policy errors, flag that prominently — it's a process anomaly worth surfacing to the user.

Don't suggest a status change (`update_threat_model_status`) unless the user asks — review is read-only by default.
