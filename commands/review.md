---
allowed-tools: Read, Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(gh repo view:*), Bash(git branch:*), AskUserQuestion
description: Deep code review - score every issue on confidence and severity, present a two-axis triage, then post on confirm
argument-hint: "[pr-number|pr-url|branch]"
disable-model-invocation: false
---

# /deep-review:review

Provide a deep code review for the given pull request. Unlike a standard review that drops everything below a confidence cutoff, this command keeps **every** scored issue, scores each on two axes (confidence that it is real, and severity if it is), presents them all in-session in a two-axis triage, recommends what to do with each, and only posts to the pull request after you confirm.

The pull request is given in `$ARGUMENTS` (a PR number, PR URL, or branch name). If empty, resolve the PR for the current branch with `gh pr list --head <branch>`.

First, read `${CLAUDE_PLUGIN_ROOT}/reference/review-shared.md`. It holds the reviewer roster, the size-scaling rules, the dedup rule, the confidence and severity rubrics, the two-axis triage table, the false-positive list, the link format, the trailer policy, and the output style. Everything below references it - do not re-derive those rules.

Then follow these steps precisely. Make a todo list first.

1. **Eligibility.** Use a Haiku agent to check whether the pull request (a) is closed, (b) is a draft, (c) does not need a code review (eg. an automated PR, or very simple and obviously fine), or (d) already has a code review from you. If any apply, stop and report why.
2. **Gather context.** Use a Haiku agent to return the file *paths* (not contents) of relevant CLAUDE.md files: the root CLAUDE.md (if any) plus any CLAUDE.md files in directories the PR modified.
3. **Summarize.** Use a Haiku agent to view the pull request and return a summary of the change. Have it also report PR size (changed files, additions, deletions) so you can pick the reviewer scale from the shared reference.
4. **Review.** Launch the reviewer roster from the shared reference (scaled to PR size) as parallel Sonnet agents. Collect every issue with its flag reason.
5. **Dedup.** Apply the dedup rule from the shared reference before scoring, so the same finding is never scored or presented twice.
6. **Score on both axes.** For each deduped issue, launch a parallel Haiku agent that receives:
   - the issue description and its flag reason,
   - the **actual diff hunk and surrounding file context** for the cited lines (fetch with `gh pr diff`),
   - the **contents** of the relevant CLAUDE.md files (not just their paths).

   The agent returns a **confidence** score (0-100, per the confidence rubric) and a **severity** (critical / major / minor, per the severity rubric). For issues flagged on CLAUDE.md grounds, the agent must confirm the CLAUDE.md actually calls out that issue specifically before scoring confidence high. Giving the agent the real code and CLAUDE.md text is what lets it verify rather than guess.
7. **Verify the impactful-but-uncertain issues.** Run the verification pass from the shared reference: every issue that is severity critical/major with confidence < 60 gets a dedicated Sonnet agent that digs into the surrounding code and resolves it to confirmed / refuted / still-uncertain, updating its confidence (and attaching a "verify: ..." note when still uncertain). Tier on the post-verification confidence.
8. **Triage, do not filter.** Keep every issue. Assign each to a tier using the two-axis triage table in the shared reference (post-verification confidence x severity). Drop only the clear false positives from the shared reference's false-positive list.
9. **Present in-session.** Before posting anything, print the triage. Lead with a verdict, then the issues:
   - **Verdict line:** a one-sentence read on the PR's health (eg. "1 blocking bug and 2 follow-ups; otherwise clean").
   - **Counts line:** `Address now: N | Address soon: N | Optional: N | Likely false positive: N`.
   - Then the issues grouped by tier (Address now first, then soon, optional, likely false positive), highest confidence first within each tier. For each issue:
     - a `[confidence]` prefix (eg. `[92]`),
     - the severity (critical / major / minor),
     - a one-line description,
     - the flag reason (CLAUDE.md adherence / bug / git history / prior PR comment / code comment),
     - the cited file+line link (shared reference link format),
     - a one-line recommendation (eg. "Fix before merge", "Follow-up issue", "Safe to ignore"). For a still-uncertain item carrying a verify note, the recommendation is "Verify <the thing>, then fix or dismiss".

   This in-session triage is the whole point of the command - the user sees everything, not just the high-confidence subset.
10. **Confirm before posting.** First, use a Haiku agent to repeat the eligibility check from step 1, to make sure the PR is still eligible. Then use `AskUserQuestion` to ask how to post:
   - `Post all tiers` - every issue, grouped by tier.
   - `Post now + soon only` - only Address now and Address soon.
   - `Post now only` - only Address now.
   - `Don't post` - print nothing to the PR; you are done.
11. **Post.** For any option other than `Don't post`, decide the trailer per the shared reference trailer policy, then use `gh pr comment` to post the selected tiers in the format below. Keep it brief, avoid emojis (except the trailer when it applies), and link/cite code per the shared reference.

## Posted comment format

Lead with the verdict and counts, put the headline tiers in a table, and wrap lower tiers in `<details>` so the comment stays scannable. Example assuming the user posted all tiers:

---

### Code review (deep-review)

<one-sentence verdict>

**Address now: 2 | Address soon: 1 | Optional: 1 | Likely false positive: 1**

**Address now**

| # | Conf | Sev | Issue | Location |
| --- | --- | --- | --- | --- |
| 1 | 92 | critical | <brief description> (CLAUDE.md says "<...>") | [file.ts#L40-L43](https://github.com/owner/repo/blob/<full-sha>/file.ts#L40-L43) |
| 2 | 78 | major | <brief description> (bug in `<snippet>`) | [api.ts#L11-L14](https://github.com/owner/repo/blob/<full-sha>/api.ts#L11-L14) |

<details>
<summary><b>Address soon</b> (1)</summary>

| # | Conf | Sev | Issue | Location |
| --- | --- | --- | --- | --- |
| 3 | 64 | minor | <brief description> (some/other/CLAUDE.md says "<...>") | [util.ts#L7-L10](https://github.com/owner/repo/blob/<full-sha>/util.ts#L7-L10) |

</details>

<details>
<summary><b>Optional / nitpick</b> (1)</summary>

| # | Conf | Sev | Issue | Location |
| --- | --- | --- | --- | --- |
| 4 | 38 | minor | <brief description> | [util.ts#L20-L23](https://github.com/owner/repo/blob/<full-sha>/util.ts#L20-L23) |

</details>

<details>
<summary><b>Likely false positive</b> (1)</summary>

| # | Conf | Sev | Issue | Location |
| --- | --- | --- | --- | --- |
| 5 | 15 | minor | <brief description> - recommend dismissing | [util.ts#L31-L34](https://github.com/owner/repo/blob/<full-sha>/util.ts#L31-L34) |

</details>

🤖 Generated with [Claude Code](https://claude.ai/code)

<sub>- This is a deep-review triage. Tiers reflect confidence and severity, not a hard pass/fail. React with 👍 if useful, 👎 otherwise.</sub>

---

- Include only the tiers the user chose to post. Omit a tier's `<details>` block if it is empty.
- Omit the `🤖 Generated with ...` trailer line on public repos (shared reference trailer policy); keep the `<sub>` footer either way.
- If every tier you chose to post is empty (eg. `Post now only` with no Address now issues), post instead:

---

### Code review (deep-review)

No issues found in the selected tiers. Checked for bugs and CLAUDE.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)

---

## Notes

- Do not check build signal or build/typecheck the app. CI runs these separately and they are not relevant to your review.
- Use `gh` to interact with GitHub, not web fetch.
- You must cite and link each issue (link the CLAUDE.md too when that is the flag reason).
- All rubrics, the reviewer roster, the false-positive list, the link format, the trailer policy, and the output style live in `${CLAUDE_PLUGIN_ROOT}/reference/review-shared.md`. Follow it; do not re-derive.
