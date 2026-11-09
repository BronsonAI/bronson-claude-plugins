---
name: fix-review
description: Act on review feedback for an open pull request — read every review comment and thread, apply the fixes, reply to each thread, push, and re-run pr-gate. Use when asked to address review comments, respond to feedback, fix review findings, or resolve a reviewer's notes on a PR.
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
gh api repos/<owner/repo>/pulls/<n>/comments --jq '.[] | {id, path, line, user: .user.login, body}'

# Top-level conversation, including pr-gate verdicts
gh api repos/<owner/repo>/issues/<n>/comments --jq '.[] | {id, user: .user.login, body}'
```

For thread state and resolution you need GraphQL:

```bash
gh api graphql -f query='
query($owner:String!, $repo:String!, $number:Int!) {
  repository(owner:$owner, name:$repo) {
    pullRequest(number:$number) {
      reviewThreads(first:100) {
        nodes { id isResolved isOutdated path line
          comments(first:20) { nodes { author { login } body } } }
      }
    }
  }
}' -f owner=<owner> -f repo=<repo> -F number=<n>
```

Skip threads already resolved. Treat outdated threads as still open unless the comment is
clearly answered by the change that outdated it.

## 3. Triage before touching code

List every item with what you intend to do, and group them. Three outcomes only:

- **Fix** — the reviewer is right. Apply it.
- **Push back** — the reviewer is wrong, or the change would cause a problem they could
  not see. Reply with the reason. Pushing back is legitimate; silently not fixing is not.
- **Out of scope** — a real issue, but not this ticket's. Offer to open a follow-up Linear
  ticket, and say so in the reply rather than just leaving it.

If more than a couple of items are "push back", show the user the triage before replying.
Disagreeing with a reviewer on their behalf is not yours to do unilaterally.

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
gh api repos/<owner/repo>/pulls/<n>/comments/<comment_id>/replies -f body="Fixed in <sha> — ..."
```

Say what changed and where, in one or two lines. "Done" is not a reply; `Fixed in a1b2c3d
— moved the tenant check above the query so the cross-tenant path cannot be reached` is.

Resolve threads you fixed. Leave threads where you pushed back **open** — resolving your
own disagreement hides it from the reviewer.

```bash
gh api graphql -f query='mutation($id:ID!) {
  resolveReviewThread(input:{threadId:$id}) { thread { isResolved } }
}' -f id=<thread_id>
```

## 6. Re-gate

Run `pr-gate` on the PR. It replaces its earlier verdict comment in place rather than
stacking a new one.

If the PR went back to draft, and the new verdict is `PASS` or `PASS WITH NOTES`, mark it
ready and re-request review:

```bash
gh pr ready <n>
gh pr edit <n> --add-reviewer BronsonAI/engineering
```

Re-requesting a human reviewer is worth doing explicitly — GitHub does not re-request on
a new push, so a review sits silently done-with until someone asks again.

## 7. Report back

Per item: what it was, what you did, where. Then the new gate verdict. Call out anything
you pushed back on and anything deferred to a follow-up ticket — those are the two things
the user needs to decide about.
