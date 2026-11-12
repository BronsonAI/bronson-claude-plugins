---
name: pr-create
description: Gate the branch with pr-gate and fix what blocks, then open the pull request as a draft and mark it ready once its checks pass, requesting the engineering team as reviewer. Use when explicitly asked to open, raise or create a PR, or to put work up for review. Not triggered by work merely being finished.
---

# pr-create

Gates the work **before** opening anything, fixes what blocks, then opens the PR as a
draft and promotes it to ready once its checks pass. A PR is never opened, and never
reaches a human reviewer, carrying a blocking finding.

**This skill is a chain of outward-facing actions** — it pushes a branch, opens a PR,
marks it ready, pings the engineering team and moves a Linear ticket. Finishing a piece of
work is not a request to do any of that. Run it only when the user asks for a PR in words,
and before the first push show what you are about to do: the branch, the base, the files
and the reviewer. One "go" covers the chain; "ok that's done" does not start it.

## 1. Preconditions

Stop and fix before opening anything:

- **Not on `main`, `develop` or `staging`.** If the work is sitting on one of those,
  stop — the change needs a branch. Do not open a PR from a protected branch.
- **Working tree clean.** Show `git status --short` — tracked changes *and* untracked
  files — and ask which belong to this ticket. Commit only those, with the `[ENG-###]`
  prefix. Never `git add -A`: scratch files, local config and secrets live in exactly the
  untracked set that a blind add would sweep in.
- **Branch name matches** `{type}/{first-name}-{KEY}-{description}`. If it does not and
  the branch has not been pushed, offer to rename it: `git branch -m <correct-name>`.
  Already pushed, or the repo has its own convention → leave it and say so in the report.
  Never rename and force-push a branch to satisfy the format.
- **Branch pushed:** `git push -u origin <branch>`.
- **CI expectations known:** run the required-checks lookup in `pr-gate` §2 against the
  base and keep the answer — a count, or *unknown* if the lookup failed — so step 5b can
  tell "nothing to wait for" from "nothing registered yet". Use that command; do not keep
  a copy of it here, which is how this step ended up on an endpoint pr-gate forbids.

## 2. Pick the base

- Feature, fix and improvement → `develop`.
- **Unless the branch is stacked.** If one of your open PRs is an ancestor of this
  branch, base on that branch instead:

```bash
git fetch origin
gh pr list --author @me --state open --json number,headRefName,baseRefName
git merge-base --is-ancestor origin/<candidate> HEAD && echo stacked
```

Exclude candidates whose head is `develop`, `main` or `backmerge/*`, and those based on
`main` — a release or back-merge PR is an ancestor of everything and is never your parent.
A candidate must also have commits that the integration branch does not (`develop`, or
the default branch in an exempt repo). If several survive,
take the nearest: the one no other candidate descends from. If that is still ambiguous,
ask rather than guessing.

- Hotfix → `main`, via the `hotfix` skill.
- Release → `main` from `develop`, via the `release` skill.

Say which base you chose and why. A wrong base is the most expensive mistake here: a
feature branch opened against `main` deploys to production on merge.

## 3. Write the PR body

Title: `[ENG-115] Case intake Postgres schema` — the Linear key, then the ticket title or
a tighter version of it.

```markdown
## ENG-115 — Case intake Postgres schema

<linear url>

## What this does
Two or three sentences in plain terms. What changed, and what it means for a user.

## Artifacts
- `docs/tickets/ENG-115/intent.md`
- `docs/tickets/ENG-115/spec.md`
- `docs/tickets/ENG-115/plan.md`

## How it was tested
What you actually ran, and the result. Not what could be run.

## Risk
Migrations, data changes, config or secret changes, anything needing a deploy order.
Write "None" if there is none.

## Stacked on
#41 — this diff includes #41's commits until #41 merges.
```

Omit the **Stacked on** section when the base is `develop`. Keep it when stacked, so the
reviewer knows the diff includes the parent's commits until the parent merges.

Do **not** retarget the child to `develop` ahead of the parent merging: the child's diff
would then show all of the parent's unmerged work, and the gate reviews the parent's code
under the child's ticket. GitHub retargets children itself when the parent merges — see
the conventions, §3, for the cases that break that and what to do about them.

If the ticket carries `prod`, the body must state a downtime line. If it carries
`destructive`, it must state the snapshot taken. Those are Linear's own rules for those
labels, and `pr-gate`'s blocking list covers them — read the ticket's labels *here*, while
writing the body, because the gate in step 4 checks this body text before the PR exists.

## 4. Gate the branch before creating anything

Run `pr-gate` in branch mode against `origin/<base>...HEAD`. **A PR is not opened on a
`CHANGES REQUESTED` verdict.**

| Verdict | Action |
|---|---|
| `APPROVED` | Open the PR (step 5) |
| `CHANGES REQUESTED` | Fix the blocking findings, commit, gate again. Loop until nothing blocks. |

Fix them here rather than opening the PR and leaving it parked as a draft. A PR that
exists is a thing teammates see, subscribe to and get notified about; one opened in a
state its own gate rejects is noise that someone else has to work out the status of. The
non-blocking findings come with you into the PR body — they are notes for the reviewer,
not reasons to stop.

If a blocking finding is one you intend to argue with rather than fix, stop and put it to
the user. Do not open the PR on the strength of your own disagreement with the gate.

## 5. Open it as a draft

```bash
gh pr create --draft --base <base> --title "<title>" --body-file <path>
```

Always `--draft` first, even on a clean gate. The draft is what lets required checks run
before a human is pulled in.

## 5b. Gate the checks

The branch gate in step 4 could not see CI — no PR existed. Now it does:

```bash
gh pr checks <n> --required --watch || true
```

Non-zero does not mean failure. When it prints "no checks reported", the answer you kept
from step 1 says which case you are in:

- **0 required** → nothing to wait for. Move on.
- **a count above 0** → they have not registered yet. Wait and run it again; never mark
  the PR ready on "no checks reported". If nothing has registered after a couple of
  minutes, say so: on some repos CI does not run until the PR is marked ready, and
  watching a draft hangs for ever.
- **unknown** → say the CI check is unverified, and why the lookup failed.

Never report an absent CI suite as a failing check. `pr-gate` §2 has the full reading.

**A required check that actually fails is blocking.** The PR stays a draft, the Linear ticket stays
`In Progress`, and you go to step 8 to report. The ticket moves to `In Review` only once
the PR is actually marked ready.

## 6. Mark ready and request review

```bash
gh pr ready <n>
gh pr edit <n> --add-reviewer BronsonAI/engineering
```

`BronsonAI/engineering` is the reviewing team. `BronsonAI/reviewers` exists but is empty —
requesting it gets you nothing. GitHub will not assign the PR author to themselves, which
is fine and not an error to report.

## 7. Close the loop in Linear

- Post the PR URL to the Linear issue as a comment or attachment.
- Move the issue to `In Review`.

## 8. Report back

Give the user the PR URL, the base branch, the gate verdict, and the reviewer requested.
If the PR is still a draft because the gate failed, lead with that — do not bury it under
the link.
