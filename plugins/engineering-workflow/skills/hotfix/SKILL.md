---
name: hotfix
description: Ship an urgent production fix — branch from main, fix, gate the branch, PR into main, then open the back-merge PR down to develop so the fix is not lost. Does not merge the production PR. Use when production is broken, when asked for a hotfix or emergency fix, or when a fix must bypass the normal develop promotion.
---

# hotfix

The one path that merges upward out of order. A hotfix branches from `main`, lands in
`main`, and is then merged back down to `develop` — because a fix that only exists in
`main` gets reverted by the next `develop` promotion.

Use it only for a production break. Anything that can wait for the normal `develop` cycle
should use `start-ticket`.

## 1. Confirm it is a hotfix

Ask, if it is not obvious: **is production broken right now?** If the answer is no, this
belongs on `develop`. Bypassing the dev environment means the fix ships to production
having never run anywhere else — that cost is only worth paying for an active break.

Then check whether `develop` is already ahead of `main` with an unrelated release in
flight. If it is, say so: the hotfix will need the back-merge in step 7 to resolve
against that work.

## 2. Ticket it

A hotfix still needs a Linear ticket — it is the audit trail for a change that skipped
the normal gates.

- An existing ticket → use its key.
- None → draft one and confirm with the user before creating it. Label it `Bug`, and add
  `prod` since it touches production by definition.
- Set it to `In Progress`.

If production is actively down, still check `list_issues` for an existing ticket, then
show a one-line draft and take a quick yes — that costs seconds. If the user is
unanswered, say in your report that the ticket is still to be created and carry on with
the fix — do not create it unconfirmed. Working without a key, use `NOTICKET` wherever
`{KEY}` appears (`hotfix/kevin-NOTICKET-token-refresh-loop`, `[NOTICKET]` commit prefixes,
`docs/tickets/NOTICKET/`) and say so in the PR body.

**Once the key exists, do not rename the branch or amend the commits.** By then the branch
is pushed, the PR is open and the back-merge may be cut from it: amending prefixes needs a
force-push, which `pr-create`, the conventions and `fix-review` all forbid on a pushed
branch, and renaming by push-and-delete closes the open PR. Instead:

- Put the real key in the **title and body of both PRs**. That is what `release`'s
  back-merge lookup searches, so the back-merge is still found under its `NOTICKET` name.
- Rename `docs/tickets/NOTICKET/` to the real key in an ordinary commit — a file move
  needs no force-push.
- Leave `backmerge/main-to-develop-NOTICKET` alone; `pr-gate`'s back-merge mode says never
  to rename it. "Unreachable" is not a state you can judge from a
prompt that has not been answered yet, and a `Bug` + `prod` ticket notifies the team with
content nobody approved. Ticketing never holds the fix; it also never happens unasked.

## 3. Branch from main

```bash
git fetch origin
git checkout -b hotfix/kevin-ENG-221-token-refresh-loop origin/main
```

`origin/main`, not `develop`, and not your local `main` — fetch first. Branching a hotfix
off a stale local `main` ships whatever else was sitting there.

Never stack a hotfix on another branch. It goes to production on merge; it carries only
its own change.

## 4. Fix it, narrowly

The diff should contain the fix and nothing else. No refactors, no drive-by cleanups, no
dependency bumps. Anything else you notice becomes a follow-up ticket.

Write the artifacts under `docs/tickets/{KEY}/` alongside the fix:

- **`intent.md`** — what broke, the blast radius, when it started, and the confirmed root
  cause. Not a suspected one.
- **`plan.md`** — the fix, and how it was verified.

Speed first, but these two are what make the skipped gates defensible afterwards.

Commit with the key prefix: `[ENG-221] Stop token refresh retry loop`.

## 5. Verify before opening

A hotfix gets less review and no dev-environment soak, so local verification carries more
weight than usual. Run the tests, and add one that fails without the fix. State what you
actually ran.

## 6. Gate the branch, then PR into main

**Write the body first.** The gate below reads it: a missing downtime line is a blocking
finding, and it cannot be checked against a body that does not exist yet.

The body must state:

- What broke and the user-visible symptom.
- The root cause.
- **A downtime line** — the expected downtime, or "No downtime expected" with the reason.
  Required on every hotfix, whether or not the ticket exists to carry the `prod` label:
  a hotfix touches production by definition, so the label is a formality and the line is
  the substance. On a `NOTICKET` hotfix say so, since the gate cannot read the label.
- How it was verified.
- Whether any data needs repair after the fix deploys.

Then gate before creating, exactly as `pr-create` does: `pr-gate` in branch mode against
`origin/main...HEAD`, reading the body text you just drafted.

| Verdict | Action |
|---|---|
| `APPROVED` | Push and open the PR |
| `CHANGES REQUESTED` | Fix the blocking findings, commit, gate again. Loop until nothing blocks. |

Urgency is the reason to gate earlier, not the reason to skip it. This branch is headed
for `main`, where the merge *is* the production deploy: a blocking finding found after
the PR is open has to be caught by the one reviewer standing between it and prod. Fixing
it here costs a minute.

```bash
git push -u origin hotfix/kevin-ENG-221-token-refresh-loop
gh pr create --draft --base main --title "[ENG-221] Stop token refresh retry loop" --body-file <path>
```

Then check CI, which the branch gate could not see — see `pr-gate` §2 for reading the
exit code, because "no checks reported" also exits non-zero and is **not** a failure. If
the base branch has required checks, wait for them to register rather than taking an
immediate "no checks reported" at face value:

```bash
gh pr checks <n> --required --watch || true
```

A required check genuinely *failing* is blocking: leave it a draft and report. A repo
with no checks configured is not. Otherwise:

```bash
gh pr ready <n>
gh pr edit <n> --add-reviewer BronsonAI/engineering
```

A hotfix still gets a human reviewer. If nobody is available and the break is severe,
that is the user's call to make explicitly — say that is what is happening rather than
quietly merging.

## 7. Back-merge into develop

**This is the step that gets forgotten, and skipping it silently reverts the fix** on the
next `develop` to `main` promotion.

**The branch must contain the fix**, so cut it from the hotfix branch, not from
`origin/main` — `main` does not have the fix until the hotfix PR merges. Cutting from
`origin/main` first gives you either `gh pr create` failing with no commits between the
branches, or a PR whose branch is a pre-fix snapshot that never picks the fix up: it
merges, the report says the back-merge is done, and `develop` still lacks the fix.

```bash
git fetch origin
git checkout -b backmerge/main-to-develop-ENG-221 hotfix/kevin-ENG-221-token-refresh-loop
git push -u origin backmerge/main-to-develop-ENG-221
gh pr create --draft --base develop --title "Back-merge ENG-221 hotfix into develop" --body-file <path>
```

A separate branch, rather than opening `main` into `develop` directly, so you can resolve
conflicts without pushing to `develop`.

**It is a snapshot.** Cut while the hotfix PR is still open, it has the hotfix as it
looked then — a commit pushed to the hotfix afterwards, in answer to review, never reaches
it. Before marking it ready, bring it up to the fix that actually shipped and check it is
there:

```bash
git fetch origin
git checkout backmerge/main-to-develop-ENG-221
git merge origin/main
git log --oneline origin/develop..HEAD
git push
gh pr ready <n>
```

If `main` carries the fix as a squash, that merge brings the squashed commit in; if as a
merge commit, the hotfix's own commits. Either way the content is what shipped. Confirm
the fix is in the diff rather than assuming the merge did it.

**Conflicts in the merge-up above are a different case.** If the hotfix changed during
review and was squash-merged, this branch holds the old version of the fix and `main`
holds the shipped one, so the same lines conflict. `main` wins there — it is what is
running in production. Do not blend the two versions.

Conflicts with `develop` are the other case: resolve them with `git merge origin/develop`
**on the back-merge branch** — never rebase it, because its commits are shared with the
hotfix PR and, once merged, with `main`. Keep both changes; if the hotfix and `develop` genuinely disagree about the same
code, stop and show the user rather than picking. Merge this PR with a merge commit, not a
squash, so the next release can match it to its hotfix.

Gate it like any other PR and put a reviewer on it. It is a real PR, not bookkeeping.
`pr-gate`'s back-merge mode skips the naming, ticket and commit-prefix checks, so a clean
back-merge gates clean.

**Gate it after the merge-up, not before.** Unlike every other PR here, this branch
changes between being opened and being ready: it is cut as a snapshot and then brought up
to what shipped. A gate run at creation reads a diff that is not the one being merged, so
run it on the PR once the merge-up is pushed and before `gh pr ready`.

**This skill does not merge the hotfix PR.** Merging to `main` is a production deploy and
the user does it. Never `gh pr merge --admin`. Open the back-merge PR as soon as the
hotfix PR is open — it can sit as a draft until the hotfix merges — so that finishing does
not depend on waiting for a merge you are not going to perform. If the hotfix has not
merged by the end of the session, say so and leave the back-merge as the named next step.

## 8. Close out

- Move the Linear ticket to `Done` once both PRs have merged, not just the first.
- If the root cause needs a durable fix beyond the patch — a test gap, missing alerting,
  a design flaw — draft a follow-up ticket and confirm it with the user.

## 9. Report back

Both PR URLs, the verdicts, and an explicit statement of whether the back-merge has
merged or is still open. A hotfix is not finished when production is fixed; it is
finished when `develop` has it too.
