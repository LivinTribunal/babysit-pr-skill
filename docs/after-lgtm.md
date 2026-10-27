# After LGTM: keeping a finished PR mergeable

The skill stops when a PR is ready. Then the PR waits for a reviewer, and while
it waits the base branch moves. In the repo this came from, 27% of merged PRs
needed someone to come back and merge main into them, and the ones that did
took a median of 44 hours from open to merge against 2 for the ones that did
not.

This doc is the design of a GitHub Actions workflow that does that coming back.
It is a design, not a drop-in file: the workflow it describes runs against one
company's labels, runners and CI, and you will want your own. Everything here
is what I would keep if I wrote it again.

## The contract

One label in, one label out, and the workflow never merges or closes anything.

| Label | Set by | Meaning |
| --- | --- | --- |
| `LGTM` | the author | The work is done. Babysitting starts. |
| `RTM` | the gate | Ready to merge: `LGTM`, an approval from someone other than the author, green CI, no conflict, no changes requested. |
| `REWORK` | the fixer | Main changed in a way that needs the author to change their own lines. `LGTM` comes off. |
| `CLOSE` | the fixer | Main already does what the PR does. `LGTM` comes off. A person closes it. |
| `babysitting` | the fixer | A fix is running on the branch. Nothing else pushes. |
| `needs-judgment` | the fixer | An attempt stopped. A person has to look. |

`RTM` is never removed by the gate. If the PR breaks after it got `RTM`, the
workflow fixes it and the label stays, so whoever merges can work from one list.

The author opts in. That matters less than you would think for adoption (see
"The button nobody pressed" below), but it matters a lot for trust: the
workflow only touches branches whose author said they are done.

## Phase 1: the gate

A cheap job with no agent and no checkout of PR code. It runs on every push to
main, every CI completion, every label, every review, every 30 minutes, and on
demand. Overlapping runs cancel each other, so a burst of merges costs one run.

One paginated GraphQL query returns every open `LGTM` PR with its
mergeability, reviews, labels, check runs and statuses. The decision per PR is
a pure function, which is what makes it testable:

1. Skip drafts, forks and dependency bumps.
2. Add `RTM` when its conditions hold.
3. Leave a PR alone while a review agent or a fixer is queued or running on it.
4. Leave mergeability `UNKNOWN` for the next run. GitHub computes it lazily
   after main moves.
5. A conflicted PR goes to the fixer.
6. Red CI gets one re-run of its failed jobs, which clears flakes without an
   agent. Still red on the same head, it goes to the fixer. Nothing is re-run
   or fixed while main itself is red.
7. A head with no CI after 30 minutes gets one CI dispatch, which uses up that
   head's re-run.

The gate also cleans up after fixer runs that died: it removes a `babysitting`
label that no queued or running fixer owns, and dispatches any review that was
waiting behind it.

## Phase 2: the fixer

An agent job, dispatched by the gate for one PR at one head.

1. **Claim.** Re-read the PR and refuse unless it is still open, same-repo, at
   the dispatched head, labelled `LGTM`, and free of every verdict and busy
   label. Write a pending commit status for this head and base pair, then add
   `babysitting`.
2. **Merge.** Merge the base commit into the head without committing. Stop if
   CI, workflow or security files would change, or if a conflicted file is a
   generated migration.
3. **Agent.** Give it the merge state, the conflicted files, and for a CI
   dispatch the failing logs. It gets edit-only tools and may touch only the
   conflicted files and the files the PR already changes. Its report starts
   with one verdict.
4. **Validate.** Reject edits outside the allowed files, untracked files,
   leftover conflict markers, binary conflicts, whitespace errors, and any
   result that drops a line the PR added or a line the base added in a
   conflicted file. A resolution that has to give up a line goes to a person.
5. **Push.** Commit a normal merge commit, re-check the PR, and push without
   force only if the remote branch is still at the dispatched head. Then
   dispatch CI for the new head.
6. **Finish.** Record the outcome on the attempt status, apply the verdict
   label, comment once per head and base pair, and remove `babysitting` in a
   step that always runs.

### The one question the fixer asks

Can this PR land without changing a line its author wrote?

- `MODERNIZE`: yes. The conflict or failure is fixable around the PR's lines.
  The fixer pushes.
- `REWORK`: no. Main deleted or moved what the PR edits, changed what a
  function it calls does, or the PR's own code fails CI. The fixer pushes
  nothing and explains the smallest rewrite.
- `CLOSE`: main already contains the behaviour.
- `NEEDS-HUMAN`: anything ambiguous.

The validator enforces the rule for `MODERNIZE`. An agent that edits a PR line
ends in `needs-judgment`, never in a push. This is the rule that lets an author
trust a bot with their branch: it may add a merge commit, it may never rewrite
what they wrote.

## Limits

- One attempt per head and base pair. The status stays on the head, so a run
  that timed out still blocks a retry of the same pair.
- Two attempts per head commit, whatever the base.
- Three fixer comments per PR per day, then `needs-judgment`.

## Review and babysit never overlap

If you also run a review agent that pushes to PR branches, give it a guard job:
when the PR carries `babysitting`, re-queue the review and exit. The fixer's
last step dispatches the queued review. The gate skips any PR whose review is
queued or running. Two bots pushing to one branch means one push gets
rejected and the review comments on code that is already gone.

## What the proven review of this workflow found

Before the workflow went live, a review that only keeps findings it can prove
with a failing test found eleven bugs. The useful ones to know about if you
build your own:

- **GitHub search lags labels.** A PR found by a label search may have lost the
  label seconds ago. Re-check the live labels in the decision function.
- **Write the "already re-ran" marker after the call succeeds**, not before, or
  one failed API call uses up a head's only retry.
- **Renames on either side break a nothing-lost check.** Compare a renamed file
  against its old path on the side that renamed it, or every line looks lost.
- **`=======` is a Markdown heading underline**, not only a conflict marker.
  Look for the full marker block.
- **A cleanup that removes a dead run's label must also start what was waiting
  on it**, or a queued review waits forever.
- **CI, workflow and security files must not drift through a merge** the bot
  pushes, even when they did not conflict.

## The runner

The gate is cheap and runs anywhere. The fixer needs an agent runner, and an
agent runner that is down looks exactly like a quiet day: dispatches queue,
then get cancelled, and nothing complains. Check that the fixer's runner label
has a live machine before you go live, and alert on fixer runs that sit queued.

## Going live

Ship the gate in log-only mode behind a repository variable. Read a day of its
decisions over your real `LGTM` PRs in the Actions log. Then flip the variable.

## The button nobody pressed

The workflow this replaced only resolved a conflict when someone added a label
asking it to. In the same period its detector ran about 2,500 times. The
resolver ran zero times. Nobody is going to come back to a PR they consider
finished to press a button. The only automation that keeps finished PRs
mergeable is the kind that starts from "finished".
