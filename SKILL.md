---
name: babysit-pr
description: Watch a pull request through CI and review bots, fix the real findings, and stop when it is ready for a human to approve. Use when the user asks to monitor, watch, or babysit a PR, or says "babysit".
---

# Babysit PR

Adapted from [t3dotgg/tstack](https://github.com/t3dotgg/tstack) (MIT, see
`LICENSE`).

This skill carries a PR from "opened" to "ready": CI green, every review bot
answered, nothing left for the author to do. It never merges. Merging is a
person's call.

If your repo also runs the after-ready workflow in `docs/after-lgtm.md`, the
skill stops at the `LGTM` label and that workflow keeps the PR mergeable until
someone merges it.

## Configure once

Fill these in for your repo and delete the ones you do not use:

- **CI workflows**: the workflows whose jobs decide "green" when the base
  branch has no required checks, for example `CI`. Checks from bots that report
  through the checks API (triage gates, review agents) usually should not gate.
- **Review agents**: bots that review the PR and may push commits to it, and
  the labels they set while queued or running.
- **Main-red signal**: how your team marks a broken main branch, for example an
  open issue with a `main-red` label.
- **Process gates**: checks that fail for a reason no code change fixes, for
  example "the PR does not link an approved issue".

## Waiting

Never write your own watcher, polling loop, or `sleep` loop. Use one watcher per
PR. Every watcher on a GitHub account shares one rate limit.

```bash
<skill dir>/scripts/pr-wait <pr> --until green --ci-workflow CI
```

`pr-wait` prints one line per real event: a check failed, all checks finished,
someone commented or reviewed, the base branch moved under files this PR
touches, the branch went BEHIND or DIRTY, the PR merged or closed. It exits 0
when the `--until` condition holds or the PR merged, 1 when it needs you, and 2
on bad input.

- `--until green` gates on the required checks. With none configured it gates
  on the jobs of each `--ci-workflow`, and with neither on every check.
- `--until ready` also waits for Greptile or Macroscope to review the head and
  for every review thread to be resolved. Use it only if you run one of them.
- In Claude Code, run it with the Monitor tool (`timeout_ms: 3600000`). Start
  it again after you push and after you handle an exit 1. If it says another
  pid already watches the PR, do not start a second one.

If your host has its own PR watcher, use that instead, never both on one PR.

When an event needs no action, end your turn without a message. Never post or
say "still waiting".

## Review agents

If a review agent may push to the branch, run `git pull` on the PR branch
before you change anything. Do not push while it is queued or running, or while
an after-ready fixer is working on the branch. If an agent stopped and asked
for a human, read its comment and tell the user. Do not clear its label
yourself.

## Findings and CI

Only act on checks and comments newer than the latest push. Verify every
finding against the source before you change code. A correctness finding you
accept gets a failing test first, then the fix. Reply with a written reason
when you dismiss one.

For a red CI job, find out which kind it is before you touch code:

- **The PR broke it.** Fix it with the smallest change.
- **Infrastructure flake** (runner died, network timeout). Re-run the failed
  jobs once with `gh run rerun <run-id> --failed`. A second failure on the same
  head is not a flake.
- **Main is red.** Do not fix main on this PR. Tell the user and wait.
- **A process gate is red.** That is a step for a person, not something to fix
  in code.

Keep the PR at the size the goal needs. Review loops grow a PR one
cheap-looking fix at a time. Fix real bugs in the intended case. Decline the
rest with a written reason and file the worthwhile ones as issues: tests
nobody needs, guards for edge cases that do not matter, extra compatibility,
small performance costs. Prefer the fix that deletes or narrows code.

## GitHub rate limits

`gh pr view` and `gh pr checks` use GraphQL. When GraphQL is used up, REST
usually still has room: `gh api repos/<owner>/<repo>/pulls/<n>` reads the PR and
`gh api repos/<owner>/<repo>/commits/<sha>/check-runs` reads the checks. Check
with `gh api rate_limit --jq .resources.graphql` and wait for `reset` with one
background command.

## Ready

The PR is ready when all of these hold on the current head:

- every gating CI job is green,
- every review agent finished on this head and its findings are fixed or
  answered,
- no agent is waiting on a human,
- no unresolved review thread, and no conflict with the base branch.

Then report it ready and stop watching. Add `LGTM` only when the user asked you
to hand the PR over. `LGTM` is the author's statement that the work is done; if
the after-ready workflow runs, it starts babysitting from there.

If an overlapping PR makes this one obsolete, stop, report it, and ask before
closing anything.
