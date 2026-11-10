---
name: hotfix
description: Ship an urgent production fix — branch from main, fix, gate, PR into main, then open the back-merge PR down to develop so the fix is not lost. Use when production is broken, when asked for a hotfix or emergency fix, or when a fix must bypass the normal develop promotion.
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
unreachable, create it, keep moving, and tell them the key as soon as they are back. Do
not let ticketing hold the fix, and do not skip it.

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

## 6. PR into main

```bash
git push -u origin hotfix/kevin-ENG-221-token-refresh-loop
gh pr create --draft --base main --title "[ENG-221] Stop token refresh retry loop" --body-file <path>
```

The body must state:

- What broke and the user-visible symptom.
- The root cause.
- **A downtime line.** The `prod` label requires it — state the expected downtime, or
  "No downtime expected" with the reason.
- How it was verified.
- Whether any data needs repair after the fix deploys.

Then run `pr-gate`. On `APPROVED`:

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

After the hotfix merges to `main`:

```bash
git fetch origin
git checkout -b backmerge/main-to-develop-ENG-221 origin/main
git push -u origin backmerge/main-to-develop-ENG-221
gh pr create --base develop --title "Back-merge ENG-221 hotfix into develop" --body-file <path>
```

Branch off `main` rather than opening `main` into `develop` directly: a separate branch
lets you resolve conflicts without pushing to `develop`.

Resolve conflicts with `git merge origin/develop` **on the back-merge branch** — never
rebase it, because its commits are already on `main`. Keep both changes; if the hotfix and
`develop` genuinely disagree about the same code, stop and show the user rather than
picking. Merge this PR with a merge commit, not a squash: a squash leaves `main`'s commits
outside `develop`'s history and the next release reports a phantom un-back-merged hotfix.

Gate it like any other PR and put a reviewer on it. It is a real PR, not bookkeeping.

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
