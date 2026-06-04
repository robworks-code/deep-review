# deep-review

A Claude Code plugin for multi-agent pull request review that **surfaces every finding** instead of hiding the low-confidence ones.

The built-in `/code-review` command runs a 5-agent review, scores each candidate issue 0-100 for confidence, then filters out everything below 80 and posts only the survivors. That hard cutoff hides medium- and low-confidence findings entirely. `deep-review` keeps the same review engine but changes what happens to the findings.

## Commands

### `/deep-review:review [pr-number|pr-url|branch]`

The default, full-fidelity review.

- Runs the same engine as `/code-review`: eligibility check, CLAUDE.md gathering, PR summary, 5 parallel reviewers, and per-issue 0-100 confidence scoring.
- **No 80 cutoff.** Every scored issue is kept and triaged into tiers by confidence:
  - **Address now** (80-100) - high confidence, real, impactful; fix before merge.
  - **Address soon** (50-79) - verified real, lower impact; worth a follow-up.
  - **Optional / nitpick** (25-49) - minor or stylistic; author's discretion.
  - **Likely false positive** (0-24) - listed briefly so nothing is hidden, but recommend dismissing.
- Presents all issues in-session, grouped by tier, each with a score, the flag reason, a cited file+line link, and a one-line recommendation.
- Then asks how much to post (all tiers / now + soon / now only / don't post) and comments on the PR only after you confirm.

### `/deep-review:auto [pr-number|pr-url|branch]`

A faster, no-confirmation variant.

- Runs the same 5-agent review but **skips the scoring step entirely** - every identified issue is treated as worth addressing.
- **Auto-applies a best-effort fix** for each issue to the working tree, then posts a summary comment to the PR. No triage, no confirmation.
- Refuses to run unless the PR's head branch is checked out locally with a clean working tree.
- Fixes are left **uncommitted** in the working tree - review with `git diff`, then commit and push yourself. The command never commits or pushes.

## Install

```bash
claude plugin marketplace add ringo380/robworks-claude-code-plugins
claude plugin install deep-review@robworks-claude-code-plugins
```

Then restart Claude Code. The commands appear namespaced as `/deep-review:review` and `/deep-review:auto`.

## Requirements

- The GitHub CLI (`gh`) authenticated against the repo you are reviewing.
- For `/deep-review:auto`: the pull request's head branch checked out locally with a clean tree.

## How it differs from `/code-review`

| | `/code-review` | `/deep-review:review` | `/deep-review:auto` |
| --- | --- | --- | --- |
| Confidence scoring | yes | yes | no |
| Drops issues below 80 | yes | no (triages instead) | n/a |
| Shows all findings in-session | no | yes | n/a |
| Confirmation before posting | no | yes | no |
| Applies fixes | no | no | yes (working tree, uncommitted) |

## License

MIT - Ryan Robson and Robworks Software LLC.
