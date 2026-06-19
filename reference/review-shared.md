# deep-review shared reference

Canonical rubrics, reviewer roster, and formatting rules shared by `/deep-review:review` and `/deep-review:auto`. Both commands read this file at start so the two stay in sync. Do not duplicate this content into the command files.

## Reviewer roster

Launch these as parallel Sonnet agents. Each returns a list of issues, and for each issue: a one-line description, the cited file and line, and the flag reason (CLAUDE.md adherence / bug / git history / prior PR comment / code comment).

- Agent #1 - CLAUDE.md compliance: audit the changes against the relevant CLAUDE.md files. CLAUDE.md is guidance for Claude as it *writes* code, so not every instruction applies during review - only flag a violation the CLAUDE.md calls out specifically.
- Agent #2 - bug scan: read the file changes and do a shallow scan for obvious bugs. Stay on the changes themselves; focus on large bugs and ignore nitpicks and likely false positives.
- Agent #3 - historical context: read the git blame and history of the modified code to find bugs in light of that history.
- Agent #4 - prior PR comments: read previous pull requests that touched these files and check for prior comments that also apply here.
- Agent #5 - code comments: read code comments in the modified files and ensure the changes comply with any guidance there.

### Scaling the roster to PR size

Probe size first (`gh pr diff <pr> --name-only` for file count, and the additions/deletions from `gh pr view <pr> --json additions,deletions,changedFiles`). Then:

- Trivial PR (<= ~15 changed lines, 1-2 files, no logic): run a single combined reviewer agent that covers all five lenses. Five agents on a one-line change is waste.
- Normal PR: the full five-agent roster above.
- Large PR (> ~800 changed lines or > ~25 files): keep the five lenses, but shard the diff by file group and run each lens per shard so no single agent has to hold the whole change. Note in-session that the review was sharded.

## Dedup (before any scoring or fixing)

Merge issues from different reviewers that point at the same line and the same root cause into one issue, keeping the clearest description and the union of flag reasons. Do this before scoring (review) or fixing (auto) so the same finding is never scored, presented, or fixed twice.

## Confidence rubric (0-100)

How likely the issue is real (not whether it matters). Give this to the scoring agent verbatim:

- 0 - Not confident at all. A false positive that doesn't survive light scrutiny, or a pre-existing issue.
- 25 - Somewhat confident. Might be real, might be a false positive; could not verify. If stylistic, it was not explicitly called out in the relevant CLAUDE.md.
- 50 - Moderately confident. Verified real, but possibly a nitpick or rare in practice.
- 75 - Highly confident. Double-checked and very likely real and hit in practice; the PR's current approach is insufficient, or it is directly named in the relevant CLAUDE.md.
- 100 - Absolutely certain. Double-checked and confirmed real, frequent in practice; evidence directly confirms it.

## Severity rubric (independent of confidence)

How much the issue matters *if it is real*. Confidence and severity are separate axes - a high-confidence typo is still low severity, and a probable-but-unverified data-loss bug is still high severity.

- critical - breaks functionality, corrupts or loses data, or is a security issue explicitly required by CLAUDE.md. Must not ship.
- major - a real bug or a clear CLAUDE.md violation with noticeable impact, but not catastrophic.
- minor - nitpick, stylistic, cosmetic, or very low-frequency edge case.

## Verification pass (resolve impactful-but-uncertain issues)

An impactful issue that hasn't been verified should not be quietly demoted and forgotten - it should be double-checked, then fixed if real or dropped if not. So before tiering, take every issue that is **severity critical or major AND confidence < 60** and give each its own verification agent (use Sonnet - this needs real digging, not a quick re-score). The agent reads the full surrounding code, traces the relevant data flow / call sites, and checks the actual behavior, then returns a revised confidence plus a verdict:

- confirmed - the issue is real. Use the raised confidence (it will now tier into Address now).
- refuted - not a real issue, or pre-existing, or already handled. Drop to Likely false positive.
- still-uncertain - genuinely ambiguous even after a real second look. Keep its mid-range confidence, and attach a one-line "verify: <the specific thing to check>" so a human (or the auto-fixer) can resolve it deterministically rather than guess.

Tier on the post-verification confidence. This makes the impactful-but-unverified cell rare and honest: most such issues resolve to Address now or Likely false positive, and the few that remain in Address soon carry an explicit "what to check" note.

## Two-axis triage

Tier each issue from both axes, using the **post-verification** confidence. Drop nothing; the lowest tier still gets listed.

| | confidence >= 60 | confidence 25-59 | confidence < 25 |
| --- | --- | --- | --- |
| severity critical/major | Address now | Address soon (verify, then fix or dismiss) | Likely false positive |
| severity minor | Address soon | Optional / nitpick | Likely false positive |

- Address now - real enough and matters; fix before merge.
- Address soon - real-but-lower-impact, or impactful-and-still-ambiguous-after-verification (carries a "verify: ..." note); worth a follow-up.
- Optional / nitpick - minor and not strongly verified; author's discretion.
- Likely false positive - listed briefly so nothing is hidden, but recommend dismissing.

## False positives (always drop these, regardless of axis)

- Pre-existing issues.
- Something that looks like a bug but is not actually a bug.
- Pedantic nitpicks a senior engineer wouldn't call out.
- Issues a linter, typechecker, or compiler would catch (imports, type errors, broken tests, formatting, newlines). CI runs these separately; do not build or typecheck yourself.
- General code-quality issues (test coverage, general security, documentation) unless explicitly required in CLAUDE.md.
- Issues called out in CLAUDE.md but explicitly silenced in the code (eg. a lint-ignore comment).
- Changes that are likely intentional or directly related to the broader change.
- Real issues on lines the user did not modify in this pull request.

## Link format

When linking to code in a posted comment, use this exact shape or the Markdown preview won't render:

`https://github.com/anthropics/claude-cli-internal/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15`

- Full git sha required. Do not use `$(git rev-parse HEAD)` inline - the comment renders as raw Markdown, so the substitution won't happen.
- Repo name must match the repo you are reviewing.
- `#` after the file name; line range `L[start]-L[end]`.
- Provide at least 1 line of context before and after the line in question (eg. for lines 5-6, link `L4-L7`).

## Trailer policy

Decide the trailer once, before posting, with `gh repo view --json visibility -q .visibility`:

- Private (or internal) repo: end the comment with the standard trailer line `🤖 Generated with [Claude Code](https://claude.ai/code)`.
- Public repo: omit that trailer line entirely. Keep the neutral `<sub>` footer described in each command.

## Output style

- Use plain hyphens (`-`) only. No em-dashes or en-dashes anywhere you write, in-session or in posted comments.
- Be concise.
- Avoid emojis except the trailer line (and only when the trailer applies, per the trailer policy).
