# babysit-pr

A Claude Code skill that keeps a pull request moving after the code is written.

Writing the code is no longer the slow part. On the repo this came from, AI
agents and developers merged 473 pull requests into main in 31 days. 130 of
them needed someone to come back and merge main into the branch, 239 times in
total, and 238 of those merges were made by a person. Those PRs took a median
of 44 hours from open to merge. The ones that never needed it took 2.

The skill handles the part before a person says "done": it watches CI and the
review bots, wakes the agent only when something real happens, fixes what a
failing test proves, answers the rest, and stops when the PR is ready. It never
merges. The design doc in `docs/after-lgtm.md` covers the part after "done": a
GitHub Actions workflow that keeps a finished PR mergeable until a person
merges it.

The full story, with numbers, is in [the blog post](https://www.flowhunt.io/blog/babysit-pr-keep-ready-pull-requests-mergeable/).

## Install

```bash
git clone https://github.com/LivinTribunal/babysit-pr-skill.git
cd babysit-pr-skill
mkdir -p ~/.claude/skills/babysit-pr
cp -r SKILL.md LICENSE scripts docs ~/.claude/skills/babysit-pr/
```

For one project instead of your whole account, use `.claude/skills/babysit-pr/`
inside the repo. `scripts/pr-wait` needs Python 3.10+ and an authenticated
`gh`.

Then open `SKILL.md` and fill in "Configure once": which workflows count as CI,
which bots review and push, how your team marks a red main, and which checks
are process gates no code change fixes.

## What is in here

| File | What it is |
|---|---|
| `SKILL.md` | The skill: one watcher per PR, how to sort a red CI job, the scope rule for review loops, the ready list |
| `scripts/pr-wait` | The watcher. Polls one PR and prints one line per real event, so the agent wakes on news and sleeps otherwise |
| `docs/after-lgtm.md` | The design of the after-ready workflow: the labels, the gate, the fixer, the rule that it never changes a line the author wrote, and the bugs a proven review found in it |

## pr-wait

```bash
scripts/pr-wait 123 --until green --ci-workflow CI
```

| Exit | Meaning |
|---|---|
| 0 | The `--until` condition holds, or the PR merged |
| 1 | It needs you: a gating check failed, the branch conflicts, the PR closed |
| 2 | Bad input, or another `pr-wait` already watches this PR |

Events are a failed check, all checks finished, a new comment or review from
someone other than the author, the base branch moving under files this PR
touches, the branch going BEHIND or DIRTY, and the PR merging or closing. A lock
stops a second watcher on the same PR, and a state file lets a restarted watcher
report what arrived while it was down.

In Claude Code run it with the Monitor tool, so each line is one wake-up. The
skill's rule is that a wake-up which needs no action ends the turn without a
message. "Still waiting" is not news.

## The rules that matter most

- **One watcher per PR.** Every watcher on a GitHub account shares one rate
  limit, and `pr-wait` refuses to start a second one on the same PR.
- **Sort the red job before touching code.** The PR broke it, the runner
  flaked, main is red, or a process gate failed. Only the first is a code fix.
  A flake gets one re-run, and a second failure on the same head is not a flake.
- **Prove before you fix.** A review finding you accept gets a failing test
  first. One you dismiss gets a written reason.
- **Keep the PR the size of its goal.** Review loops grow a PR one
  cheap-looking fix at a time. Fix real bugs, decline the rest in writing, and
  file the worthwhile ones as issues.
- **Ready is a list, and LGTM is a person's word.** Green CI, review bots
  finished on this head, no unresolved thread, no conflict. Then report it and
  stop. The agent adds `LGTM` only when told to hand the PR over.

## After LGTM

The skill stops at ready because the next wait is for a person, and that wait
is where PRs go stale. `docs/after-lgtm.md` is the design for a workflow that
watches every `LGTM` PR, merges main in when it conflicts, re-runs red CI once
and then fixes it, and adds `RTM` when someone else approves. It never merges,
never closes, and never changes a line the author wrote: a resolution that
would have to is handed back with `REWORK` instead.

It is a design rather than a file to copy because it runs against one
company's labels, runners and CI. The doc says what to keep and which mistakes
to skip.

## Companion skills

[architect-skill](https://github.com/LivinTribunal/architect-skill) has the
expensive model plan and review while a cheaper worker writes the code.
[grain-skill](https://github.com/LivinTribunal/grain-skill) makes that worker
mirror what the repo already has. babysit-pr is what happens after either of
them opens the PR.

## Licence

MIT. Adapted from the `babysit-pr` skill in
[t3dotgg/tstack](https://github.com/t3dotgg/tstack), also MIT.
