---
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(gh repo view:*), Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git symbolic-ref:*), Bash(git stash create), Bash(make:*), Bash(npm run:*), Bash(npm test:*), Bash(pnpm:*), Bash(yarn:*), Bash(cargo:*), Bash(go test:*), Bash(./gate.sh:*), Bash(./scripts/gate.sh:*), Bash(./check.sh:*), Bash(scripts/check:*), Bash(./scripts/check:*), Bash(claude plugin validate:*)
description: Fast deep review - skip scoring, auto-apply fixes for every issue found, run the local gate, then report (and comment if there is a PR)
argument-hint: "[pr-number|pr-url|branch]"
disable-model-invocation: true
---

# /deep-review:auto

A faster, no-confirmation variant of `/deep-review:review`. It **skips the confidence/severity scoring entirely**, treats every identified issue as worth addressing, **auto-applies a best-effort fix** for each to the working tree, verifies the result against the repo's local gate, and reports. No triage, no confirmation prompt. It posts a PR summary comment **only when there is a PR** - see Step 0's modes.

A pull request may be given in `$ARGUMENTS` (a PR number, PR URL, or branch name). If empty, try to resolve one for the current branch with `gh pr list --head <branch>`. A PR is optional: with none, the command reviews the branch and reports in-session.

First, read `${CLAUDE_PLUGIN_ROOT}/reference/review-shared.md`. It holds the reviewer roster, the size-scaling rules, the dedup rule, the false-positive list, the link format, the trailer policy, and the output style. Everything below references it - do not re-derive those rules. (This command ignores the confidence/severity rubrics and the triage table, since it does not score.)

Then follow these steps precisely. Make a todo list first.

## Step 0 - Establish mode and baseline

This command edits source files. It does not require a PR, and it does not require a clean tree - but it must know which situation it is in before it touches anything.

```bash
cur=$(git branch --show-current)
git status --porcelain
gh repo view --json defaultBranchRef -q .defaultBranchRef.name
```

Resolve the default branch with `gh` as shown - **do not use `origin/HEAD`.** That ref is only populated by `git clone` and is absent on any repo created locally and pushed afterwards, where `git rev-parse --abbrev-ref origin/HEAD` and `git merge-base HEAD origin/HEAD` both die with `fatal: ambiguous argument`. If `gh` is unavailable, fall back in order: `git symbolic-ref refs/remotes/origin/HEAD`, then `origin/main`, then `origin/master`.

**Pick the mode:**

- Resolve the PR (via `gh pr view` / `gh pr list --head "$cur"`) and read its `headRefName`.
- **PR mode** - a PR exists and `$cur` is its head branch. Review scope is the PR diff. You will post a summary comment in Step 5.
- **Wrong branch** - a PR exists but `$cur` is *not* its head branch. Stop and report which branch to check out. This is the one case that still refuses: editing files while the PR's code is not what is checked out fixes the wrong thing.
- **Branch mode** - no PR for this branch. Do not refuse. Review scope is the diff against the merge-base with the default branch (`git merge-base HEAD "origin/<default>"`, using the default branch resolved above). If `$cur` *is* the default branch, review scope is the uncommitted working-tree diff plus any local commits ahead of the remote. Report in-session only - **do not** post a comment, there is nothing to post to.

**Record the dirty baseline (all modes):**

- Capture the exact file list from `git status --porcelain` **before any edit** and keep it. This is the user's pre-existing uncommitted work.
- A dirty tree is no longer a refusal. It does mean your fixes will land interleaved with the user's own changes in the same `git diff`, so the Step 5 summary must list the pre-existing dirty files explicitly, under the heading "already modified before this run - fixes are interleaved here". Without that list the user cannot tell your edits from their own.
- **Split the baseline against the review scope** - this distinction is what makes a dirty-tree run useful rather than inert:
  - *Dirty **and** in review scope* - **fixable.** When you are reviewing uncommitted work (branch mode on the default branch), the dirty files *are* the thing under review. Refusing to touch them would mean reviewing everything and fixing nothing.
  - *Dirty **and** outside review scope* - **never touch.** This is unrelated work in flight: a half-finished edit elsewhere in the repo that has nothing to do with the change under review. Never revert, stash, checkout, or discard it. If the only correct fix for an issue would rewrite a region there, **skip it** and record "would overwrite unrelated uncommitted work in `<file>`" as the reason.
- **Never run `git checkout`, `git restore`, `git reset`, `git clean`, `git stash`, or `git revert` on any file - ever, and with no exceptions.** This command edits forward only. When Step 4 needs to back out one of your own fixes, undo it with an `Edit` that restores the original text, which you have because Step 3 recorded it. A git-level restore cannot distinguish your fix from the user's uncommitted work and will take both.
- Pass this same ban to every agent you dispatch (see the shared reference). It is not enough for the top-level command to observe it.

**Snapshot before you start (safety net):**

```bash
git stash create
```

Run this once, after recording the baseline and before applying any fix, and keep the returned SHA. `git stash create` builds a dangling commit that captures the current tree **without modifying the working tree or the stash list** - it is purely a snapshot, not a stash push. If fixes are ever lost, that SHA is what recovers them (`git show <sha>`), so print it in the Step 5 summary. If the tree is clean it returns nothing, which is fine - there is nothing to lose yet.

**Eligibility (PR mode only):** run a Haiku agent - if the PR is (a) closed, (b) a draft, (c) doesn't need review, or (d) already reviewed by you, do not proceed. In branch mode there is no eligibility check to run; skip it.

## Step 1 - Gather context

1. Use a Haiku agent to list the file paths of relevant CLAUDE.md files: the root CLAUDE.md (if any) plus any in directories the review scope modified. Read their contents - you will apply their guidance when fixing.
2. Summarize the change and measure its size (changed files, additions, deletions) so you can pick the reviewer scale from the shared reference. In PR mode use a Haiku agent with `gh pr view` / `gh pr diff`; in branch mode use `git diff --stat <merge-base>...HEAD` plus the uncommitted diff. The size drives the roster scale identically either way.
3. Determine the verification signal per the shared reference's "Verification signal" section: check for `.github/workflows`, and if absent, identify the repo's local gate command now. Find it before you start editing - discovering there is no gate *after* applying ten fixes is too late to be useful.
4. **If a gate exists, run it now, before any edit, and keep the result.** This is the baseline Step 4 needs to tell "your fix broke it" apart from "it was already broken." A gate result with no baseline to compare against cannot be attributed and is close to worthless.

## Step 2 - Review (no scoring)

Launch the reviewer roster from the shared reference (scaled to the size measured in Step 1) as parallel Sonnet agents. Collect every issue with its flag reason, then apply the dedup rule from the shared reference.

**The reviewers are read-only** - paste the tree-mutation ban from the shared reference into every reviewer prompt verbatim. A subagent does not inherit this command's `allowed-tools`; it gets its own, typically broad, tool set. The prompt-level ban is therefore the only thing standing between a helpful agent and a `git checkout -- .` that silently erases every fix applied so far. Reviewers report; only Step 3 edits, and it edits serially.

**Do not score or rank.** Every surviving issue is treated as actionable. Still drop the clear false positives from the shared reference's false-positive list - skipping scoring does not mean fixing non-issues. Note that when the repo has no CI, that list no longer discards compiler-class findings; they are actionable here like anything else.

## Step 3 - Auto-apply fixes (no confirmation)

Work through the deduped issues one at a time. For each:

- **Double-check before fixing.** Confirm the issue is real by reading the surrounding code and tracing the relevant behavior (this is the auto-fix equivalent of the shared reference's verification pass - it applies to every issue here, since there are no scores). Fix only what you can confirm.
- **If you dispatch an agent to do that double-check, paste the shared reference's tree-mutation ban into its prompt.** Fixes from earlier iterations are sitting uncommitted in the tree right now; a verification agent that "resets to a clean state" destroys them, and `git status` will look fine afterwards. This is the single most damaging failure mode of this command.
- **Record every fix as you apply it** - file, the exact region changed, and a one-line description. Step 4's integrity check needs this list, and it is also the only record that survives if something does clobber the tree.
- If confirmed, apply a minimal, targeted fix with Edit/Write that resolves the issue while respecting the relevant CLAUDE.md and surrounding code style.
- If the issue cannot be confirmed or fixed safely and mechanically (refuted on a closer look, ambiguous intent, needs a design decision, would touch lines outside the review scope, or risks breaking behavior), **skip it** and record a one-line reason. Do not guess at risky changes.
- If the fix would rewrite a region of a file that is dirty **and outside the review scope**, **skip it** per Step 0 and record that reason. Dirty files that are *inside* the review scope are fair game - fixing them is the point.
- Keep edits confined to files the review scope already modified. Do not refactor unrelated code.

Do **not** commit or push. Fixes are left uncommitted for the user to review with `git diff`, then commit and push themselves.

## Step 4 - Verify: local gate, then fix integrity

An auto-applied fix that breaks the build is the failure mode this command is most likely to produce, so verify before reporting rather than asserting success.

- **CI present:** skip the gate run and say so in the summary ("not gated locally; CI covers it"). Still run the integrity check below.
- **No CI, gate found** (from Step 1.3): run it and report the actual result.
- **No CI, no gate found:** say so plainly in the summary. Do not imply the change was verified when nothing verified it.

**You need a pre-fix baseline to attribute a failure.** A gate result after fixing is uninterpretable on its own - a repo whose gate was already red will look like you broke it. So when a gate exists, run it **once at the end of Step 1, before any edit**, and keep that result. Then:

- Baseline red, still red: report "gate was already failing before this run" and name the pre-existing failure. Do not attribute it to yourself and do not revert anything on its account.
- Baseline green, now red: one of your fixes broke it. Identify which, undo that one **with an Edit restoring the original text** (never a git restore - see Step 0), record it as skipped with reason "reverted: broke the local gate", and re-run the gate once.
- Baseline red, now green: say so. You fixed something that was already broken.

**Integrity check (always, even when CI covers the gate).** Step 0's cleanliness check is a precondition, not an invariant - nothing re-checks it, so a mid-run clobber goes unnoticed. Before reporting, walk the Step 3 fix list and confirm **each fix is still present in the file**. For every fix, re-read the region and verify the change is actually there.

- All present: proceed to Step 5.
- Any missing: something reverted the tree mid-run. Do **not** report those fixes as applied - that is the failure mode that makes this command lie. Recover from the `git stash create` SHA taken in Step 0, re-apply the missing fixes, and re-run the integrity check. State plainly in the summary that a mid-run revert was detected and recovered.

Never report a fix as applied on the strength of the Edit having succeeded. Edit success means the text changed at the time; it does not mean the change survived to the end of the run.

## Step 5 - Report

**Always report in-session**, in the format below. **Post it as a PR comment only in PR mode** - in branch mode there is no PR, so the in-session report is the whole deliverable and you must not invent somewhere to post it.

In PR mode, decide the trailer per the shared reference trailer policy, then use `gh pr comment` to post. Keep it brief, avoid emojis (except the trailer when it applies), and link/cite code per the shared reference link format. Use this format:

---

### Deep review (auto-fix)

Applied <N> fixes to the working tree (uncommitted - review with `git diff`, then commit and push):

| # | Fix | Reason | Location |
| --- | --- | --- | --- |
| 1 | <brief description> | CLAUDE.md / bug / history / comment | [file.ts#L40-L43](https://github.com/owner/repo/blob/<full-sha>/file.ts#L40-L43) |
| 2 | <brief description> | bug | [api.ts#L11-L14](https://github.com/owner/repo/blob/<full-sha>/api.ts#L11-L14) |

Local gate: <one of - `scripts/check` passed (was green before this run) | `scripts/check` FAILED, see below | `scripts/check` was already failing before this run | not gated locally, CI covers it | no CI and no local gate found, these fixes are unverified>
Integrity: <one of - all <N> fixes verified present | <M> fixes were reverted mid-run and re-applied, recovered from snapshot `<sha>`>

<details>
<summary>Could not auto-fix <M> issues (left for manual review)</summary>

- <brief description> - <one-line reason it was skipped>

</details>

<details>
<summary>Already modified before this run - fixes are interleaved here (<K> files)</summary>

- `path/to/file.ts`

</details>

<!-- trailer: add the Claude Code trailer line here ONLY if the repo is confirmed PRIVATE per the shared reference trailer policy; otherwise omit entirely -->

<sub>- Fixes are uncommitted in the author's working tree. Review before merging.</sub>

---

- Omit the "Could not auto-fix" block if nothing was skipped.
- Omit the "Already modified before this run" block if the Step 0 baseline was empty (clean tree). Never omit it when the tree was dirty - it is the only thing that lets the user separate your edits from their own.
- The "Local gate" and "Integrity" lines are never omitted. If nothing verified the fixes, the gate line says so; the integrity line always states whether every recorded fix was confirmed still present.
- Report the `git stash create` snapshot SHA in-session (not in the posted comment) so the user can recover if anything went wrong.
- The trailer is opt-in, not opt-out: include the `🤖 Generated with ...` line ONLY if `gh repo view --json visibility` positively returned PRIVATE or INTERNAL (shared reference trailer policy). On a public repo, an errored check, or a check you skipped, omit it. Keep the `<sub>` footer in all cases.
- If no fixes were applied and nothing was skipped, post instead (real newlines, not escaped):

---

### Deep review (auto-fix)

No issues found. Checked for bugs and CLAUDE.md compliance.

<!-- trailer: add the Claude Code trailer line here ONLY if the repo is confirmed PRIVATE per the shared reference trailer policy; otherwise omit entirely -->

---

- Also report the same summary in-session, and remind the user the fixes are uncommitted. In branch mode, report only in-session.

## Notes

- **Do not assume CI exists.** Apply the shared reference's "Verification signal" rule: if the repo has no `.github/workflows`, compiler-class findings are real findings rather than false positives, and Step 4 runs the local gate. A repo with no CI is usually a repo that verifies locally on purpose, not an unverified one.
- Use `gh` to interact with GitHub, not web fetch.
- The reviewer roster, size scaling, dedup rule, false-positive list, link format, trailer policy, and output style live in `${CLAUDE_PLUGIN_ROOT}/reference/review-shared.md`. Follow it; do not re-derive.
