---
name: fix-review
description: Act on review feedback for an open pull request — read every comment and thread, show the triage, apply the fixes, reply to each thread, push, re-gate, and re-request the reviewer. Use when asked to address review comments, respond to feedback, fix review findings, or resolve a reviewer's notes on a PR.
---

# fix-review

Turns review feedback into commits, replies and a fresh verdict. Works on human reviews
and on `pr-gate` verdicts alike.

## 1. Resolve the PR

A number, URL, or nothing (use the current branch's PR):

```bash
gh pr view --json number,title,headRefName,baseRefName,isDraft,reviewDecision
```

Make sure the branch is checked out and up to date with the remote before changing
anything — someone may have pushed to it.

## 2. Collect every piece of feedback

Three separate sources, and missing one is how a comment gets dropped:

```bash
# Review summaries and their state
gh pr view <n> --json reviews

# Inline comments on the diff
gh api --paginate repos/<owner/repo>/pulls/<n>/comments --jq '.[] | {id, in_reply_to_id, path, line, user: .user.login, body}'

# Top-level conversation, including pr-gate verdicts
gh api --paginate repos/<owner/repo>/issues/<n>/comments --jq '.[] | {id, user: .user.login, body}'
```

For thread state and resolution you need GraphQL. Ask for `databaseId` on the comments:
the REST reply endpoint needs it, and without it there is no way to get from a thread back
to the comment you must reply to.

```bash
gh api graphql --paginate -f query='
query($owner:String!, $repo:String!, $number:Int!, $endCursor:String) {
  repository(owner:$owner, name:$repo) {
    pullRequest(number:$number) {
      reviewThreads(first:100, after:$endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes { id isResolved isOutdated path line
          comments(first:20) { nodes { databaseId author { login } body } } }
      }
    }
  }
}' -f owner=<owner> -f repo=<repo> -F number=<n>
```

`--paginate` needs both the `$endCursor` variable and `pageInfo { hasNextPage endCursor }`
to walk the pages; without them it fetches one page of 100 and says nothing about the
rest. A long review reaches that: this plugin's own PR passed 48 threads in three rounds.
Count the threads you got and say the number, so a truncation shows up as a number that
looks wrong rather than as feedback that quietly never arrived.

Skip threads already resolved. Treat outdated threads as still open unless the comment is
clearly answered by the change that outdated it.

## 3. Triage before touching code

List every item with what you intend to do, and group them. Three outcomes only:

- **Fix** — the reviewer is right. Apply it.
- **Push back** — the reviewer is wrong, or the change would cause a problem they could
  not see. Reply with the reason. Pushing back is legitimate; silently not fixing is not.
- **Out of scope** — a real issue, but not this ticket's. Offer to open a follow-up Linear
  ticket, and say so in the reply rather than just leaving it.

**Always show the user the triage before posting anything.** Not just the push-backs —
the whole list, grouped, one line each. Replies go out under the user's name, and a reply
that concedes or disputes a point on their behalf is theirs to approve. There is no
volume threshold: three items get the same treatment as thirty.

Do not escalate what is already decided. A convention the user chose is not an open
question because a reviewer dislikes it — push back on their behalf and say why. Bring
them only what genuinely turns on their judgement.

## 4. Apply the fixes

- One commit per coherent group of feedback, prefixed `[ENG-###]`. A commit per comment
  is noise; one commit for twelve unrelated fixes is unreviewable.
- **If feedback changes the approach, update `docs/tickets/{KEY}/plan.md` in the same
  commit.** `pr-gate` checks the diff against the plan, so a stale plan turns into a
  finding on the next pass.
- If feedback expands scope beyond the Linear ticket, stop and raise it. The ticket is
  the scope of record; a reviewer asking for more is a ticket conversation, not a quiet
  commit.
- Do not rebase or force-push a branch under review unless the user asks. It detaches
  comments from their lines and makes the reviewer re-read everything.

```bash
git push
```

## 5. Reply to every thread

Every item gets a reply, including the ones you fixed — a fix with no reply leaves the
reviewer re-checking it manually.

```bash
# reply in an inline thread
# <comment_id> is the databaseId of the thread's FIRST comment, not the thread id
gh api repos/<owner/repo>/pulls/<n>/comments/<comment_id>/replies -f body="Fixed in <sha> — ..."
```

Say what changed and where, in one or two lines. "Done" is not a reply; `Fixed in a1b2c3d
— moved the tenant check above the query so the cross-tenant path cannot be reached` is.

**Findings that arrived as top-level comments have no thread to reply in.** A reviewer
puts them there whenever the line is not in the diff, and they are easy to answer nowhere
and call the round complete. Answer them in one comment that names each finding:

```bash
gh pr comment <n> --body-file <path>
```

They cannot be resolved, so the only record that they were handled is that comment.

Resolve threads you fixed. Leave threads where you pushed back **open** — resolving your
own disagreement hides it from the reviewer.

```bash
gh api graphql -f query='mutation($id:ID!) {
  resolveReviewThread(input:{threadId:$id}) { thread { isResolved } }
}' -f id=<thread_id>
```

## 6. Re-gate

Run `pr-gate` on the PR. On a PR you authored it reports the verdict in the terminal and
posts nothing — it is a preflight check. On someone else's it asks before submitting, and
a re-gate posts a new review, because a review's state cannot be changed once submitted.

On `APPROVED`, mark it ready if it is a draft:

```bash
gh pr ready <n>
```

On `CHANGES REQUESTED`, leave it as it is and say what is still open. If the PR was
marked ready earlier in this run, offer `gh pr ready <n> --undo`.

## 7. Re-request the reviewer — always

**This step is not optional and does not depend on the gate verdict.** Re-request every
reviewer who left feedback, from step 2:

```bash
gh pr edit <n> --add-reviewer <each reviewer from step 2>
```

GitHub does not re-request on a push. A reviewer who left comments sees their review as
done and will not look again until they are asked, so a run that fixes everything and
skips this leaves the PR sitting exactly where it was, with nobody waiting on anything.

It runs even when you fixed nothing and pushed back on all of it — then it matters more,
because the reviewer has to answer the push-back before the PR can move. It runs on a
`CHANGES REQUESTED` gate too: your own gate's opinion is not what the reviewer is waiting
on. The only reason to skip it is the user telling you to.

Check it landed rather than assuming:

```bash
gh pr view <n> --json reviewRequests --jq '[.reviewRequests[].login]'
```

## 8. Report back

Per item: what it was, what you did, where. Then the new gate verdict. Call out anything
you pushed back on and anything deferred to a follow-up ticket — those are the two things
the user needs to decide about.
