---
name: pr-review
description: Advisory review of a pull request on any code host (GitHub, GitLab, Bitbucket). Reviews only changed lines against the repo's rules and posts inline comments plus one summary comment. Invoke as "/pr-review <PR id or URL>", manually or from a CI pipeline.
---

# PR Review

Review the PR passed as the argument and post findings to it. Comment only — never modify files, push, approve, or change PR state, and never fail a CI step.

Use whatever tools reach the PR's host: an MCP server, the host's CLI (`gh`, `glab`), or its REST API. Read the repo, source and target branch from the PR (or CI env vars such as `$BITBUCKET_PR_ID`, `$GITHUB_REPOSITORY`, `$CI_MERGE_REQUEST_IID`). If you can't tell which PR or host, stop and say so.

## Steps

1. **Read the rules** — root `CLAUDE.md`, `~/.claude/CLAUDE.md` if present, and every file in `.claude/rules/`.
2. **Get the diff** and list the added/changed lines per file. Skip lock files, generated code, and build output.
3. **Review each changed hunk**, reading the surrounding file for context. Look for:
   - `bug` — logic errors, unhandled null/undefined, broken async, regressions
   - `security` — injection, leaked secrets, unsafe input handling
   - `rule` — anything that breaks the rules from step 1
   - `library` — misuse of the project's component library (see below)
4. **Verify each finding** before keeping it: re-read the code and confirm the problem is real and on a changed line. Drop anything you can't explain concretely.
5. **Check existing comments** — on a re-run, don't repost a finding already left by a previous run.
6. **Post** (see Output).

**Severity:** `high` = bug, security issue, or data loss. `medium` = clear rule break or likely wrong behavior. `low` = style/readability.

## Component library checks

Only if the project has a design system or shared component library. Flag on changed lines:

- **high** — a handler written for `onChange(event, value)` passed to a component that emits `onChange(value)`, or the reverse. It silently does nothing.
- **high** — a tooltip/popover that clones its child to attach a ref, wrapped directly around a toggle/checkbox that forwards its ref to a hidden input. The click area disappears.
- **medium** — consumer CSS overriding a library component's own styling (padding, radius, colour, border, shadow, hover/disabled states). Consumer CSS is for layout between components only.
- **medium** — a hand-rolled component the library already ships, a new import from a library the project is moving away from, or a raw library component where the project has a wrapper for it.
- **medium** — `error={Boolean(x)}` or `error=""` on a field whose `error` prop displays its value.
- **medium** — a typography `variant` that renders a heading tag, with no tag override, on content that isn't a heading.
- **low** — hardcoded `px`/hex where a design token with the same value exists.

A difference between the library and an old design is not a finding. A consumer patching around the library is.

## Output

**Inline comment** per finding, anchored to the changed line (`path` + new-file line). Skip findings you can't anchor.

```
**[severity · category]** <one-line issue>
<optional: suggested fix, ≤3 lines>
```

**One summary comment:**

```
## 🤖 AI PR Review (advisory)

**Verdict:** <looks good | changes suggested>
**Findings:** <H> high · <M> medium · <L> low

<optional: 1–2 sentences on the overall theme>
```

No findings → summary only, verdict "looks good", plus `✅ no issues found on changed lines`.

No praise, no restating the diff, and no repeating inline findings in the summary.
