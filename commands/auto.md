---
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(gh repo view:*), Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*)
description: Fast deep review - skip scoring, auto-apply fixes for every issue found, then post a summary comment
argument-hint: "[pr-number|pr-url|branch]"
disable-model-invocation: true
---

# /deep-review:auto

A faster, no-confirmation variant of `/deep-review:review`. It **skips the confidence/severity scoring entirely**, treats every identified issue as worth addressing, **auto-applies a best-effort fix** for each to the working tree, and posts a summary comment. No triage, no confirmation prompt.

The pull request is given in `$ARGUMENTS` (a PR number, PR URL, or branch name). If empty, resolve the PR for the current branch with `gh pr list --head <branch>`.

First, read `${CLAUDE_PLUGIN_ROOT}/reference/review-shared.md`. It holds the reviewer roster, the size-scaling rules, the dedup rule, the false-positive list, the link format, the trailer policy, and the output style. Everything below references it - do not re-derive those rules. (This command ignores the confidence/severity rubrics and the triage table, since it does not score.)

Then follow these steps precisely. Make a todo list first.

## Step 0 - Preconditions (refuse rather than guess)

This command edits source files, so the PR's head branch must be checked out locally with a clean tree.

```bash
cur=$(git branch --show-current)
git status --porcelain
```

- Resolve the PR (via `gh pr view` / `gh pr list --head "$cur"`) and read its `headRefName`.
- If the current branch is **not** the PR's head branch, stop and report which branch to check out. Do not edit files.
- If `git status --porcelain` shows a **dirty** working tree, stop and report. Do not edit files on top of uncommitted changes.
- Run a Haiku agent for the eligibility check: if the PR is (a) closed, (b) a draft, (c) doesn't need review, or (d) already reviewed by you, do not proceed.

## Step 1 - Gather context

1. Use a Haiku agent to list the file paths of relevant CLAUDE.md files: the root CLAUDE.md (if any) plus any in directories the PR modified. Read their contents - you will apply their guidance when fixing.
2. Use a Haiku agent to view the PR and return a summary plus its size (changed files, additions, deletions) so you can pick the reviewer scale from the shared reference.

## Step 2 - Review (no scoring)

Launch the reviewer roster from the shared reference (scaled to PR size) as parallel Sonnet agents. Collect every issue with its flag reason, then apply the dedup rule from the shared reference.

**Do not score or rank.** Every surviving issue is treated as actionable. Still drop the clear false positives from the shared reference's false-positive list - skipping scoring does not mean fixing non-issues.

## Step 3 - Auto-apply fixes (no confirmation)

Work through the deduped issues one at a time. For each:

- **Double-check before fixing.** Confirm the issue is real by reading the surrounding code and tracing the relevant behavior (this is the auto-fix equivalent of the shared reference's verification pass - it applies to every issue here, since there are no scores). Fix only what you can confirm.
- If confirmed, apply a minimal, targeted fix with Edit/Write that resolves the issue while respecting the relevant CLAUDE.md and surrounding code style.
- If the issue cannot be confirmed or fixed safely and mechanically (refuted on a closer look, ambiguous intent, needs a design decision, would touch lines outside the PR's scope, or risks breaking behavior), **skip it** and record a one-line reason. Do not guess at risky changes.
- Keep edits confined to files the PR already modified. Do not refactor unrelated code.

Do **not** commit or push. Fixes are left uncommitted for the user to review with `git diff`, then commit and push themselves.

## Step 4 - Post a summary comment

Decide the trailer per the shared reference trailer policy, then use `gh pr comment` to post. Keep it brief, avoid emojis (except the trailer when it applies), and link/cite code per the shared reference link format. Use this format:

---

### Deep review (auto-fix)

Applied <N> fixes to the working tree (uncommitted - review with `git diff`, then commit and push):

| # | Fix | Reason | Location |
| --- | --- | --- | --- |
| 1 | <brief description> | CLAUDE.md / bug / history / comment | [file.ts#L40-L43](https://github.com/owner/repo/blob/<full-sha>/file.ts#L40-L43) |
| 2 | <brief description> | bug | [api.ts#L11-L14](https://github.com/owner/repo/blob/<full-sha>/api.ts#L11-L14) |

<details>
<summary>Could not auto-fix <M> issues (left for manual review)</summary>

- <brief description> - <one-line reason it was skipped>

</details>

🤖 Generated with [Claude Code](https://claude.ai/code)

<sub>- Fixes are uncommitted in the author's working tree. Review before merging.</sub>

---

- Omit the "Could not auto-fix" block if nothing was skipped.
- Omit the `🤖 Generated with ...` trailer line on public repos (shared reference trailer policy); keep the `<sub>` footer either way.
- If no fixes were applied and nothing was skipped, post instead (real newlines, not escaped):

---

### Deep review (auto-fix)

No issues found. Checked for bugs and CLAUDE.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)

---

- Also report the same summary in-session, and remind the user the fixes are uncommitted.

## Notes

- Do not check build signal or build/typecheck the app. CI runs these separately.
- Use `gh` to interact with GitHub, not web fetch.
- The reviewer roster, size scaling, dedup rule, false-positive list, link format, trailer policy, and output style live in `${CLAUDE_PLUGIN_ROOT}/reference/review-shared.md`. Follow it; do not re-derive.
