# deep-review shared reference

Canonical rubrics, reviewer roster, and formatting rules shared by `/deep-review:review` and `/deep-review:auto`. Both commands read this file at start so the two stay in sync. Do not duplicate this content into the command files.

## Reviewer roster

Launch these as parallel Sonnet agents. Each returns a list of issues, and for each issue: a one-line description, the cited file and line, and the flag reason (CLAUDE.md adherence / bug / git history / prior PR comment / code comment).

**Reviewers are strictly read-only** - see the tree-mutation ban below, which applies to them and to every other agent either command dispatches. Applying fixes is the calling command's job, done serially after review, never a reviewer's.

## Dispatched agents must never mutate the working tree

This applies to **every** agent either command spawns: reviewers, eligibility checkers, context gatherers, scorers, and - most importantly - verification agents. State it verbatim in every agent prompt:

> You are read-only. Do not use Edit, Write, or NotebookEdit. Do not run any command that mutates the working tree - in particular never `git checkout`, `git restore`, `git reset`, `git stash`, `git clean`, or `git revert`. Inspect and report; return findings as data.

This is not theoretical. Under `/deep-review:auto` the calling command applies fixes to the working tree and leaves them uncommitted while the loop continues, so a verification agent that "resets to a clean state" with `git checkout -- .` **destroys every fix applied so far** - and `git status` afterwards looks clean, so the loss is silent. This has actually happened: a completed fix was applied, confirmed present, then found reverted, with nothing in `git status` to indicate anything was wrong.

Note that a subagent does **not** inherit the calling command's `allowed-tools` restrictions - it gets its own tool configuration, typically a broad one. So the fact that `git checkout` is absent from `auto.md`'s `allowed-tools` does not prevent a subagent from running it. The prompt-level ban above is the only thing that does.

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

An impactful issue that hasn't been verified should not be quietly demoted and forgotten - it should be double-checked, then fixed if real or dropped if not. So before tiering, take every issue that is **severity critical or major AND confidence < 60** and give each its own verification agent (use Sonnet - this needs real digging, not a quick re-score). **Verification agents are the highest-risk agents in this plugin** - they are the ones tempted to "get back to a known state" - so the tree-mutation ban above is mandatory in their prompts. The agent reads the full surrounding code, traces the relevant data flow / call sites, and checks the actual behavior, then returns a revised confidence plus a verdict:

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
- Issues a linter, typechecker, or compiler would catch (imports, type errors, broken tests, formatting, newlines) - **but only when something else actually runs those checks.** See "Verification signal" below; on a repo with no CI and no local gate, these are real findings and must be reported, not dropped.
- General code-quality issues (test coverage, general security, documentation) unless explicitly required in CLAUDE.md.
- Issues called out in CLAUDE.md but explicitly silenced in the code (eg. a lint-ignore comment).
- Changes that are likely intentional or directly related to the broader change.
- Real issues on lines the user did not modify in this pull request.

## Verification signal (do not assume CI exists)

Compiler-class findings are only safe to drop if some other process catches them. Never assume that process is CI. Determine which of these the repo actually has, once, before triaging or fixing:

```bash
[ -d .github/workflows ] && find .github/workflows -maxdepth 1 \( -name '*.yml' -o -name '*.yaml' \)
```

(Do not probe with a bare `ls .github/workflows/*.yml`. Under zsh a glob matching nothing is a `no matches found` **error**, not an empty result, and `2>/dev/null` does not suppress it because the shell fails before `ls` runs - inside an `&&` chain it aborts the whole step. Quote the pattern and let `find` do the matching.)

- **CI present** (one or more workflow files): treat linter/typechecker/compiler-class findings as covered and drop them per the false-positive list. Do not build or typecheck yourself.
- **No CI**: nothing downstream catches them. **Do not drop them** - report them like any other finding, flag reason `build`. A repo can deliberately have no CI and rely on a local gate instead, which means these findings are the author's only signal, and silently discarding them is the worst possible behavior.

When there is no CI, also look for the repo's local gate - the single command that runs format/lint/typecheck/test. Check, in order:

1. **What CLAUDE.md says.** If a CLAUDE.md names the project's verification command, that is the gate; use it verbatim and stop looking. This outranks every heuristic below - a project that documents its own gate has already answered the question.
2. A `Makefile` target (`check`, `test`, `lint`).
3. `package.json` scripts (`check`, `typecheck`, `lint`, `test`).
4. An executable at `scripts/check`, `scripts/gate.sh`, `gate.sh`, or `check.sh` - note these are often extensionless, so do not search for `*.sh` alone.
5. The language default (`cargo clippy && cargo test`, `go test ./...`).
6. For a Claude Code plugin repo, identified by a `.claude-plugin/plugin.json`: `claude plugin validate .`. Be honest about what that last one proves: it validates the manifest, not behavior. A prompt-only plugin has no executable gate, and the summary should say the fixes are unverified rather than treating a passing manifest check as verification. Whether you may *run* it depends on the command - `/deep-review:review` is read-only and must only report the gate command it found; `/deep-review:auto` edits source and must actually run it. If you cannot identify a gate, say so plainly rather than implying the change was verified.

## Link format

When linking to code in a posted comment, use this exact shape or the Markdown preview won't render:

`https://github.com/anthropics/claude-cli-internal/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15`

- Full git sha required. Do not use `$(git rev-parse HEAD)` inline - the comment renders as raw Markdown, so the substitution won't happen.
- Repo name must match the repo you are reviewing.
- `#` after the file name; line range `L[start]-L[end]`.
- Provide at least 1 line of context before and after the line in question (eg. for lines 5-6, link `L4-L7`).

## Trailer policy

**Default to no trailer.** Add one only after positively confirming the repo is private:

```bash
gh repo view --json visibility -q .visibility
```

- Output is exactly `PRIVATE` or `INTERNAL`: you may end the comment with `🤖 Generated with [Claude Code](https://claude.ai/code)`.
- **Anything else - `PUBLIC`, an error, an empty result, or a command you did not actually run: no trailer.** Omit it.

The templates below are written trailer-free for this reason. The trailer is something you *add* on a confirmed-private repo, never something you *remove* on a public one. The earlier formulation ("include it, omit on public") failed open: it put an AI-attribution line on a public repo whenever the visibility check was skipped, misread, or errored, and that is exactly the case that must not leak. Keep the neutral `<sub>` footer in all cases.

## Output style

- Use plain hyphens (`-`) only. No em-dashes or en-dashes anywhere you write, in-session or in posted comments.
- Be concise.
- Avoid emojis except the trailer line (and only when the trailer applies, per the trailer policy).
