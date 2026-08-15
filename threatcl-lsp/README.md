# threatcl-lsp — Claude Code plugin

`threatcl-lsp` wires the [`threatcl lsp`](https://github.com/threatcl/threatcl/blob/main/docs/lsp.md)
language server into Claude Code, so the agent gets **live diagnostics** on HCL
threat models as it edits them: syntax errors, unknown blocks/attributes, missing
required attributes, and invalid enum values (e.g. an unrecognised risk
`likelihood`) are pushed into Claude's context after every edit. Claude can then
catch and fix invalid HCL in the same turn it writes it.

This is an **opt-in companion** to the `threatcl-cloud` plugin — install it only
if you want the trade-off described below.

## Install

```
claude plugin marketplace add threatcl/claude-plugin
claude plugin install threatcl-lsp@threatcl
```

## Prerequisites

- **`threatcl` CLI ≥ 0.5.0** on your `PATH`. The `lsp` subcommand ships in 0.5.0.
  This plugin only tells Claude Code *how* to launch the server (`threatcl lsp`);
  it does not bundle the binary.
  - macOS / Linux: `brew install threatcl`
  - Go: `go install github.com/threatcl/threatcl/cmd/threatcl@latest`
  - Releases: <https://github.com/threatcl/threatcl/releases>
- Verify with `threatcl lsp -help`.

If `threatcl` isn't on `PATH`, Claude Code's `/plugin` **Errors** tab will show
`Executable not found in $PATH`.

## What you get

Once installed, Claude Code starts `threatcl lsp` over stdio and feeds it every
`*.hcl` file the agent opens or edits. The agent gets:

- **Diagnostics** (the main event) — pushed automatically after each edit.
- **Hover** and **document symbols / outline** — available to the agent on demand.

(Completion and formatting are part of `threatcl lsp` for interactive editors, but
aren't consumed by Claude Code's LSP tool. Go-to-definition / references aren't
implemented by the server yet.)

## The `*.hcl` caveat — read before installing

Claude Code matches language servers on a file's **final extension segment only**.
A file named `model.tm.hcl` resolves to `.hcl`, so there is **no way** to scope
this server to threatcl's preferred `*.tm.hcl` suffix the way Neovim, Helix, and
VS Code can. `threatcl-lsp` therefore claims **all** `*.hcl` files. That means:

- It also attaches to **Terraform / Packer / Nomad** HCL and will report spurious
  "unknown block" diagnostics on those files.
- If you also install a Terraform/HCL LSP plugin, **only one server can own
  `.hcl`** — whichever Claude Code loads first wins, and the other is silently
  dropped for `.hcl` files.
- It attaches to **`.threatcl-ci.hcl`**, the config file for
  [`threatcl/drift-action`](https://github.com/threatcl/drift-action), and reports
  it as an invalid threat model. It isn't one — it's a CI config that happens to
  be HCL. The action's own model discovery deliberately excludes it, and globs
  `*.tm.hcl` rather than bare `*.hcl`, precisely so a CI config is never mistaken
  for a model. The language server has no equivalent filename skip yet; until it
  does, ignore diagnostics on that file. The fix belongs in `threatcl lsp`, not
  here — `.lsp.json` can map `.hcl → threatcl` and nothing finer.

This is exactly why the language server is a **separate, opt-in plugin** rather
than bundled into `threatcl-cloud`: cloud users who also edit Terraform shouldn't
be forced into this trade-off.

**Rule of thumb:** install `threatcl-lsp` if your repos are primarily threatcl HCL
threat models. Skip it — and rely on `threatcl validate` / `threatcl cloud
validate` instead — if you mostly edit Terraform or other HCL dialects in the same
Claude sessions.

## Known limitation

`threatcl lsp`'s `-config` / `.hcltmrc` overrides (custom information
classifications, initiative sizes, etc.) do **not** yet affect LSP diagnostics —
the server uses threatcl's built-in spec enum defaults. See the
[LSP docs](https://github.com/threatcl/threatcl/blob/main/docs/lsp.md#known-limitations)
for the full list.

## License

MIT — see [`LICENSE`](https://github.com/threatcl/claude-plugin/blob/main/LICENSE).
