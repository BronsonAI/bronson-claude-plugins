---
name: pr-create
description: Open a pull request as a draft, gate it with pr-gate, then mark it ready and request the engineering team as reviewer. Use when asked to open, raise or create a PR, to put work up for review, or when finishing a ticket's implementation.
---

# pr-create

Opens the PR as a draft, runs `pr-gate` against it, and only promotes it to ready once
the verdict allows. A PR never reaches a human reviewer without having been gated first.

## 1. Preconditions

Stop and fix before opening anything:

- **Not on `main`, `develop` or `staging`.** If the work is sitting on one of those,
  stop — the change needs a branch. Do not open a PR from a protected branch.
- **Working tree clean.** Uncommitted changes belong in a commit first, with the
  `[ENG-###]` prefix.
- **Branch name matches** `{type}/{first-name}-{KEY}-{description}`. If it does not and
  the branch has not been pushed, rename it: `git branch -m <correct-name>`.
- **Branch pushed:** `git push -u origin <branch>`.
- **CI expectations known:** check whether required checks exist on the base branch so
  you know what the gate will be waiting on.

## 2. Pick the base

- Feature, fix and improvement → `develop`.
- **Unless the branch is stacked.** If one of your open PRs is an ancestor of this
  branch, base on that branch instead:

```bash
gh pr list --author @me --state open --json number,headRefName
git merge-base --is-ancestor origin/<candidate> HEAD && echo stacked
```

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
#41 — retarget this PR to `develop` before #41 merges.
```

Omit the **Stacked on** section when the base is `develop`. Keep it when stacked, because
merging a parent *and deleting its branch* closes the child PR, and GitHub then refuses
both to reopen it and to change its base.

If the ticket carries `prod`, the body must state a downtime line. If it carries
`destructive`, it must state the snapshot taken. Those are Linear's own rules for those
labels, and `pr-gate` fails the PR without them.

## 4. Open it as a draft

```bash
gh pr create --draft --base <base> --title "<title>" --body-file <path>
```

Always `--draft` first, even when you are confident. The draft is what gives the gate
somewhere to post before a human is pulled in.

## 5. Gate it

Run `pr-gate` against the new PR number. Then:

| Verdict | Action |
|---|---|
| `PASS` / `PASS WITH NOTES` | Continue to step 6 |
| `CHANGES REQUESTED` / `BLOCK` | **Leave it as a draft.** Report the findings and offer `fix-review`. Do not request a reviewer. |

If required checks are still running, wait for them rather than promoting the PR on a
partial verdict. Say that you are waiting.

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
