---
name: start-ticket
description: Start work on a Linear ticket — read the ticket, create the correctly named branch, write the SDLC artifacts (intent, spec, plan) for the ticket type, and move it to In Progress. Use when told to start, pick up or begin a ticket, when given an ENG-### key to work on, or when asked to start a bug, feature or improvement.
---

# start-ticket

Takes a Linear ticket from `Todo` to a branch with an approved plan on it. One skill for
all three types — the ticket's type label decides the branch prefix and which artifacts
get written.

This is the Plan, Design and Build-planning end of the AI-native SDLC. It stops at the
human gate: **the plan is presented for approval, not executed.**

## 1. Resolve the ticket

Accept an `ENG-###` key, a Linear URL, or a description of the work.

- Key or URL → read it, including comments. `get_issue` does not return the discussion;
  call `list_comments` too. Screenshots in the description are markdown images — pass
  them through `extract_images` to actually see them.
- A description with no ticket → **draft the ticket and confirm it with the user before
  creating anything.** Show the title, description, type label and priority, and wait.
  Never create a Linear ticket silently.
- Check `list_issues` for a matching ticket first, so you do not create a duplicate.

Read the ticket as the definition of scope. Do not infer requirements that are not in it;
list what is missing instead.

## 2. Determine the type

From the ticket's label — `Bug`, `Feature` or `Improvement`:

| Label | Branch type | Artifacts |
|---|---|---|
| `Feature` | `feature` | `intent.md`, `spec.md`, `plan.md` |
| `Bug` | `fix` | `intent.md`, `plan.md` — add `spec.md` only if the fix changes observable behaviour |
| `Improvement` | `feature`, `fix` or `chore`, whichever fits the change | `plan.md` |

`chore` is tooling, dependencies, CI and docs. If an `Improvement` could be more than
one branch type, ask.

No type label on the ticket → ask which it is and offer to add the label. Do not guess
from the title.

If this is an urgent production break that cannot wait for the normal path, stop and use
`hotfix` instead — it branches from `main`, not `develop`.

## 3. Create the branch

```
{type}/{first-name}-{KEY}-{description}
```

`first-name` is the author's first name in lowercase — derive it from
`git config user.name`, not from the GitHub login. `KEY` is uppercase. `description` is
kebab-case, five words at most.

**Pick the base before branching.** If you have an open *feature* PR in this repo, branch
off its head, not `develop`:

```bash
git fetch origin
gh pr list --author @me --state open --json number,headRefName,baseRefName
```

Never stack on a PR whose base is `main` — a hotfix or a release. When that parent merges,
GitHub retargets your child to `main`, and the feature ships straight to production
without ever soaking in dev. Skip `backmerge/*` heads for the same reason. If more than
one candidate survives, take the nearest ancestor and ask when it is ambiguous.

Successive PRs tend to touch the same shared files — `package.json`, lockfiles, CI config
— so two branches cut from `develop` conflict the moment one merges. Stacking serialises
that. Tell the user which base you chose and why.

**Resolve the integration branch first — do not assume `develop` exists.** Most repos in
the org have no `develop` branch, and `git checkout -b ... origin/develop` fails outright
there with `fatal: 'origin/develop' is not a commit`:

```bash
git fetch origin
if git ls-remote --exit-code --heads origin develop >/dev/null 2>&1; then
  BASE=develop
else
  BASE=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)   # exempt repo
fi
echo "basing on $BASE"
```

Never create a `develop` branch to make a repo fit the convention, and never improvise a
base silently — say which one you chose. A stacked branch uses the parent's head instead
of `$BASE`.

```bash
git checkout -b feature/kevin-ENG-115-case-intake-postgres "origin/$BASE"   # or origin/<parent-branch>
```

## 4. Write the artifacts

Under `docs/tickets/{KEY}/`. Seed `intent.md` from the Linear description — the ticket is
authoritative for scope, so intent restates it rather than reinventing it.

**`intent.md`** — the problem, not the solution.
```markdown
# ENG-115 — Case intake Postgres schema

**Linear:** <url> · **Type:** Feature · **Author:** Kevin

## Problem
...

## Who it affects
...

## Outcome wanted
...

## Constraints
...

## Open questions
- [ ] ...
```

For a `Bug`, `intent.md` leads with reproduction and root cause instead:
```markdown
## Reproduction
1. ...

## Expected vs actual
...

## Root cause
...
```

Do not write a root cause you have not confirmed in the code. If you have not found it
yet, say so and find it before writing `plan.md`.

**`spec.md`** — the design. Behaviour, interfaces and schemas, data and migrations,
failure modes, security (authz, PHI/PII handling, tenant isolation), and UX notes.
Anything that outlives this ticket belongs in Notion instead, linked from here.

**`plan.md`** — the implementation plan.
```markdown
# ENG-115 — Plan

## Files
| File | Change |
|---|---|
| `apps/api/src/intake/schema.ts` | New Kysely table definitions |
| ... | ... |

## Order of work
1. ...

## Risks
- ...

## Test strategy
- ...
```

Name real files. Read the code first — a plan listing files that do not exist is worse
than no plan.

## 5. Stop at the gate

Commit the artifacts on their own:

```bash
git add docs/tickets/ENG-115
git commit -m "[ENG-115] Add intent, spec and plan"
```

Then **present the plan and wait for approval before implementing.** Show the user the
file list, the order of work, the risks, and any open questions from intent. That
approval is the human gate; do not roll past it in the same turn.

If the ticket has unresolved open questions that change the implementation, raise them
now rather than picking an answer.

## 6. Move the ticket

**After the approval in step 5, not before.** This is bookkeeping that announces to the
team that work has started, so it waits for the gate like everything else.

Read the issue's current status and assignee first. Then:

- Unassigned → assign it to the user. Already assigned to someone else → leave it and say so.
- `Todo` or `Backlog` → set `In Progress`.
- Any other status, or a reassignment → confirm with the user before changing it. Someone
  moved it there deliberately.
- Blocking open questions → `Blocked` instead, with a comment saying why. Show the comment
  before posting it.

## 7. While building

- Keep commits prefixed `[ENG-115]`.
- Deviating from the plan is expected. Update `plan.md` **in the same commit** that
  deviates, so the diff and the plan still describe the same work — `pr-gate` checks them
  against each other.
- When the work is done, use `pr-create`.
