---
description: Analyze code (file, directory, or git diff) and suggest threats from the Threatcl library plus novel threats worth modeling
argument-hint: <path-or-diff-range>
---

You are analyzing code at `$ARGUMENTS` for security implications and analyzing and comparing against the current threatcl `.hcl` threat model file. The argument may be a file path, a directory, or a git ref range like `HEAD~3..HEAD` or `main...feature-branch`.

## 1. Read the target and existing Threatcl HCL threat model

- If `$ARGUMENTS` looks like a git ref range (contains `..`), run `git diff $ARGUMENTS` and analyze the diff.
- If it's a directory, list files and focus on code likely to have security relevance (handlers, auth, data access, network, crypto, file I/O, deserialization, templating).
- If it's a single file, read it directly.

If the argument is empty, run `git diff HEAD~1..HEAD` against the most recent commit and analyze that.

Also seek out and review the local threatcl `.hcl` file in this repo. If this doesn't existing, you can still continue, but you can suggest to the user to run `/threat-hcl-new`

## 2. Identify security-relevant patterns

Skim for: authentication, authorization, session handling, password/secret handling, cryptography, input parsing, SQL/NoSQL queries, deserialization, file paths, subprocess execution, network calls, templating/rendering, logging of sensitive data, rate limiting, and anything that crosses a trust boundary.

For each pattern you find, note the file and line number — every threat you suggest must be tied to a concrete code location.

## 3. Match against the library

Use the `list_library_items` MCP tool (or `threatcl cloud library threats` or `threatcl cloud library controls`) with keywords drawn from what you found. For each promising library item, fetch its full detail (`get_library_item` or `threatcl cloud library threat-ref`) so you can recommend it accurately.

Prefer `ref`-based suggestions over generated threats whenever a library item fits — that keeps the org's library doing its job.

## 4. Identify novel threats

If you found a security-relevant pattern that no library item covers, propose a new threat. Don't propose duplicates of existing library items; if it's almost-but-not-quite a match, note that and suggest extending the library item rather than forking.

## 5. Output

Format:

```
# Threat analysis: <path or diff>

## Code surface examined
- <file>:<line range> — <what it does>
…

## Library threats that apply
- T-<ref> — <name>
  - Why: <pattern at <file>:<line> matches this threat>
  - HCL snippet:
    ```hcl
    threat "<short name>" { ref = "T-<ref>" }
    ```
…

## Novel threats worth modeling
- <name>
  - Why: <pattern at <file>:<line>>
  - STRIDE: <one or more>
  - Impacts: <list>
  - HCL snippet:
    ```hcl
    threat "<short name>" {
      description = "<one sentence>"
      impacts     = [<…>]
      stride      = [<…>]
      # control "..." { ... }  # add controls when known
    }
    ```
…

## Suggested next steps
1. <e.g. "Open the team's API threat model and add the SQL injection threat">
…
```

Don't write to any HCL file unless the user asks. The default is "report and let the user decide what to add where."
