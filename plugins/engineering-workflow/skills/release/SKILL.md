---
name: release
description: Promote develop to main — collect what is shipping into a changelog, open the release PR, gate it, and move the shipped Linear tickets to Done. Use when asked to cut a release, promote, ship to production, or merge develop into main.
---

# release

Opens the `develop` to `main` promotion PR. Promotion is by merging the whole branch
upward, so this ships everything currently on `develop`.

`staging` exists between the two in some repos and is not in use today. Treat
`develop` to `main` as the only promotion path until that changes.

**This skill does not merge.** Authorising a production deploy is the user's call.

## 1. Check develop is shippable

```bash
git fetch origin
gh run list --branch develop --limit 5
```

- Latest `develop` CI must be green. A red `develop` means something already merged
  broken — find out what before promoting, and say so rather than opening the PR.
- No open PR already promoting `develop` to `main`. If one exists, update it instead of
  opening a second.

```bash
gh pr list --base main --head develop --state open
```

## 2. Work out what is shipping

```bash
gh api repos/<owner/repo>/compare/main...develop --jq '{ahead: .ahead_by, behind: .behind_by, commits: [.commits[] | {sha: .sha[0:7], message: (.commit.message | split("\n")[0]), author: .commit.author.name}]}'
```

- `ahead_by` of 0 → nothing to ship. Say so and stop.
- `behind_by` above 0 → `main` has commits `develop` does not, which means a hotfix was
  never back-merged. **Stop and fix that first** with a back-merge PR, or the promotion
  will revert it.

Pull the `ENG-###` keys out of the commit messages and read those tickets for their titles
and type labels. A commit with no key still ships — list it under its raw message rather
than dropping it.

## 3. Flag what needs attention

Before writing anything, check the shipping set for:

- Tickets labelled **`prod`** — each needs its own downtime line.
- Tickets labelled **`destructive`** — each needs a snapshot confirmed taken.
- **Migrations** in the diff. List them explicitly with their order, and say whether they
  are reversible.
- **Config or secret changes** that have to be in place in production before the deploy.
  A deploy that needs a secret nobody has set is the most common way a green release
  breaks production.

```bash
gh api repos/<owner/repo>/compare/main...develop --jq '.files[].filename' | grep -iE 'migration|schema|\.env|secret|config'
```

## 4. Write the release PR

Title: `Release: develop to main — 2026-11-09` (today's date).

```markdown
## Shipping to production

12 commits · 8 tickets · 2026-11-09

### Features
- **ENG-115** Case intake Postgres schema — #42
- **ENG-118** Provider search filters — #44

### Fixes
- **ENG-207** Duplicate webhook retry — #47

### Improvements
- **ENG-203** Drop the unused legacy host route — #45

### Needs attention before merge
- `prod` — **ENG-115** adds three tables. No downtime expected; the migration is additive
  and runs before the app rollout.
- **Migrations**, in order: `0031_case_intake`, `0032_provider_index`. Both reversible.
- **Config** — `INTAKE_EXPORT_BUCKET` must exist in prod before deploy.

### Deploy order
1. Apply migrations
2. Deploy API
3. Deploy web

### Rollback
...how to get back, and whether the migrations block a rollback...
```

Group by the Linear type label. Omit empty sections. The **Needs attention** section is
the part a release manager actually reads — if it is empty, write "Nothing flagged" so it
is clear you checked rather than skipped it.

## 5. Open and gate

```bash
gh pr create --base main --head develop --title "<title>" --body-file <path>
```

Not a draft — a release PR has nothing to iterate on, and leaving it draft stops the
required checks from mattering.

Run `pr-gate` on it. For a release PR the gate is narrower than for a feature PR: the code
was already gated on the way into `develop`. What matters here is the P1 pass over the
combined diff, the migration and config flags from step 3, and whether `main` is behind.

Then:

```bash
gh pr edit <n> --add-reviewer BronsonAI/engineering
```

## 6. Hand it over

Give the user the PR URL, the count of commits and tickets, the gate verdict, and the
**Needs attention** list verbatim. Then stop.

Say explicitly that merging deploys to production and that you have not merged it.

## 7. After it merges

Only once the user confirms the merge:

- Move every shipped Linear ticket to `Done`.
- Check that CI/CD actually deployed: `gh run list --branch main --limit 3`. A merged
  release PR with a failed deploy workflow is worse than an unmerged one, because
  everyone assumes it shipped.
- Report the deploy run's result. Do not report a release as shipped on the strength of
  the merge alone.
