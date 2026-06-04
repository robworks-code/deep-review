---
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*)
description: Fast deep review - skip scoring, auto-apply fixes for every issue found, then post a summary comment
argument-hint: "[pr-number|pr-url|branch]"
disable-model-invocation: false
---

# /deep-review:auto

A faster, no-confirmation variant of `/deep-review:review`. It **skips the confidence scoring step entirely**, treats every identified issue as worth addressing, **auto-applies a best-effort fix** for each one to the working tree, and posts a summary comment to the pull request. There is no triage and no confirmation prompt.

The pull request is given in `$ARGUMENTS` (a PR number, PR URL, or branch name). If empty, resolve the PR for the current branch with `gh pr list --head <branch>`.

To do this, follow these steps precisely. Make a todo list first.

## Step 0 - Preconditions (refuse rather than guess)

This command edits source files, so the pull request's head branch must be checked out locally with a clean tree.

```bash
cur=$(git branch --show-current)
git status --porcelain
```

- Resolve the PR (via `gh pr view` / `gh pr list --head "$cur"`) and read its `headRefName`.
- If the current branch is **not** the PR's head branch, stop and report which branch to check out. Do not edit files.
- If `git status --porcelain` shows a **dirty** working tree, stop and report. Do not edit files on top of uncommitted changes.
- Also run a Haiku agent for the standard eligibility check: if the PR is (a) closed, (b) a draft, (c) doesn't need review, or (d) already reviewed by you, do not proceed.

## Step 1 - Gather context

1. Use a Haiku agent to list the file paths of relevant CLAUDE.md files: the root CLAUDE.md (if any) plus any CLAUDE.md files in directories the PR modified. Read their contents - you will apply their guidance when fixing.
2. Use a Haiku agent to view the pull request and return a summary of the change.

## Step 2 - Review (no scoring)

Launch 5 parallel Sonnet agents to independently review the change. Each returns a list of issues with the reason each was flagged:
   a. Agent #1: Audit changes for CLAUDE.md compliance. CLAUDE.md is guidance for writing code, so not all instructions apply during review.
   b. Agent #2: Shallow scan of the file changes for obvious bugs. Focus on the changes themselves and on large bugs; ignore nitpicks and likely false positives.
   c. Agent #3: Read git blame and history of the modified code to find bugs in light of that history.
   d. Agent #4: Read previous pull requests touching these files and check for applicable prior comments.
   e. Agent #5: Read code comments in the modified files and ensure the changes comply with any guidance there.

Collect all issues and lightly dedup (merge issues that point at the same line and root cause). **Do not score or rank them** - every surviving issue is treated as actionable.

Still drop clear false positives using the list below - skipping scoring does not mean fixing non-issues.

## Step 3 - Auto-apply fixes (no confirmation)

Work through the deduped issues one at a time. For each:

- Apply a minimal, targeted fix with Edit/Write that resolves the issue while respecting the relevant CLAUDE.md and surrounding code style.
- If the issue cannot be fixed safely or mechanically (ambiguous intent, needs a design decision, would touch lines outside the PR's scope, or risks breaking behavior), **skip it** and record a one-line reason. Do not guess at risky changes.
- Keep edits confined to files the pull request already modified. Do not refactor unrelated code.

Do **not** commit or push. Fixes are left uncommitted in the working tree for the user to review with `git diff`, then commit and push themselves.

## Step 4 - Post a summary comment

Use `gh pr comment` to post a summary. Keep it brief, avoid emojis (except the trailer), and link/cite code. Use this format:

---

### Deep review (auto-fix)

Applied <N> fixes to the working tree (uncommitted - review with `git diff`, then commit and push):

1. <brief description of fix> (<reason: CLAUDE.md / bug / history / comment>)

<link to file and line with full sha1 + line range for context>

2. <brief description of fix>

<link to file and line with full sha1 + line range for context>

Could not auto-fix <M> issues (left for manual review):

- <brief description> - <one-line reason it was skipped>

🤖 Generated with [Claude Code](https://claude.ai/code)

<sub>- Fixes are uncommitted in the author's working tree. Review before merging.</sub>

---

- If no fixes were applied and nothing was skipped, post: `### Deep review (auto-fix)\n\nNo issues found. Checked for bugs and CLAUDE.md compliance.\n\n🤖 Generated with [Claude Code](https://claude.ai/code)`
- Also report the same summary in-session, and remind the user the fixes are uncommitted.

## False positives (still drop these)

- Pre-existing issues
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues that a linter, typechecker, or compiler would catch (imports, type errors, broken tests, formatting, newlines). CI runs these separately.
- General code quality issues (test coverage, general security, documentation) unless explicitly required in CLAUDE.md
- Issues called out in CLAUDE.md but explicitly silenced in the code (eg. a lint ignore comment)
- Changes that are likely intentional or directly related to the broader change
- Real issues on lines the user did not modify in their pull request

## Notes & link format

- Do not check build signal or attempt to build or typecheck the app. CI runs these separately.
- Use `gh` to interact with Github, not web fetch.
- When linking to code in the comment, use the exact format: https://github.com/anthropics/claude-cli-internal/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15
  - Full git sha required (do not use `$(git rev-parse HEAD)` inline - the comment renders as raw Markdown)
  - Repo name must match the repo you're reviewing
  - `#` after the file name, line range `L[start]-L[end]`, with at least 1 line of context before and after

## Output style
- Use plain hyphens (`-`) only. No em-dashes or en-dashes.
- Be concise.
