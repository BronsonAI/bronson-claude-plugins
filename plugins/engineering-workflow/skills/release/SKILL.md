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
gh api repos/<owner/repo>/compare/main...develop --jq '{ahead: .ahead_by, commits: [.commits[] | {sha: .sha[0:7], message: (.commit.message | split("\n")[0]), author: .commit.author.name}]}'
```

- `ahead_by` of 0 → nothing to ship. Say so and stop.
- `behind_by` above 0 is **not** evidence of an un-back-merged hotfix. Every past
  promotion leaves a commit on `main` that `develop` lacks, and in a squash-only repo it
  stays that way for good.

  Commit topology cannot answer this: a squashed promotion commit has one parent, so
  `--no-merges` keeps it, and its patch-id matches no single `develop` commit, so
  `--cherry-pick` keeps it too. Any check built on `git log` gives the same false signal
  `behind_by` does. **Ask about PRs instead**, which is merge-strategy independent:

```bash
# when develop was last promoted
LAST=$(gh pr list --base main --head develop --state merged --limit 1 \
  --json mergedAt --jq '.[0].mergedAt // ""')

# hotfixes merged into main since then
LAST="$LAST" gh pr list --base main --state merged --limit 100 \
  --json number,headRefName,title,mergedAt \
  --jq '.[] | select(.headRefName | startswith("hotfix/")) | select(.mergedAt > env.LAST) | [.number, .headRefName, .title] | @tsv'

# for each one, the back-merge it should have, looked up by its exact branch name
gh pr list --base develop --state merged \
  --head backmerge/main-to-develop-ENG-221 --json number,title
```

  **Only hotfixes since the last release.** Anything older has already been through a
  promotion; checking it again stops every future release on history. FOIAger's
  `hotfix/prod-env`, from before this convention existed, would otherwise block all of
  them. With no earlier release, `LAST` is empty and every hotfix is in scope.

  For each hotfix, take the key from its **branch name** — `ENG-221`, or the
  `NOTICKET-<yyyymmddhhmm>` placeholder for one shipped before its ticket existed — and
  look up `backmerge/main-to-develop-<that>` with `--head`. `hotfix` names the back-merge
  from the same key and never renames either branch, so the two always match exactly.

  An exact head match is the point. A full-text `--search` for the key also matches any
  feature PR that merely mentions it; listing back-merges and filtering in jq loses older
  ones to `--limit`, which applies before the filter. Neither happens with `--head`.

  A hotfix whose branch carries neither an `ENG-###` nor a `NOTICKET-<…>` placeholder was
  made outside `hotfix`, so there is no name to look its back-merge up by. That is
  *unknown*, not missing: put it under **Needs attention** and ask the user to confirm its
  fix is in `develop` before going on.

  No merged PR for that head → the hotfix was never back-merged: stop. For a placeholder
  hotfix, also report the real key from its title, or that the title still has none —
  the ticket was never created. Everything else on `main` is normal promotion history.
  Merging `develop` into `main` is a three-way merge and does not by itself revert a
  `main`-only commit.

The compare API caps at 250 commits and 300 files; past that it truncates silently. Use
`git log origin/main..origin/develop` for the commit list and say so explicitly if either
cap is near.

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
git diff --name-only origin/main...origin/develop | grep -iE 'migration|schema|seed|\.env|secret|config|terraform|\.tf$'
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

Not a draft — a release PR has nothing to iterate on. (Required checks do run on a
draft; a draft simply cannot be merged. The reason to skip draft here is that there is no
iteration step to protect, not that CI would be skipped.)

Run `pr-gate` on it in release mode. **The release PR is a tracking artifact, not a
review.** Every feature in it was gated and approved on the way into `develop`; re-reviewing
them here would find nothing new and would put a second veto on work that already passed.

So the gate does not re-read the features. It looks only for what is new at promotion
time, which is the handful of things no single feature PR could have seen:

- The migration and config or secret flags from step 3, across the combined diff.
- A merged hotfix with no matching back-merge (step 2). Not `behind_by`, and not any
  `git log` form of it — step 2 says why.
- Two changes that are each fine alone and conflict together: competing migrations on one
  table, a config key set twice.

Nothing else blocks a release. A convention finding on the release branch is noise by
construction — the head is `develop` and the commits are merges.

Then, **only on `APPROVED`**:

```bash
gh pr edit <n> --add-reviewer BronsonAI/engineering
```

On `CHANGES REQUESTED`, do not request a reviewer. Report the blocking findings and stop:
a production PR carrying a known blocker is the one place a reviewer's rubber stamp costs
the most.

## 6. Hand it over

Give the user the PR URL, the count of commits and tickets, the gate verdict, and the
**Needs attention** list verbatim. Then stop.

Say explicitly that merging deploys to production and that you have not merged it.

## 7. After it merges

Only once the user confirms the merge:

Verify the deploy **before** touching Linear — a ticket moved to `Done` on a failed
deploy is a lie that nobody re-checks.

```bash
SHA=$(gh pr view <n> --json mergeCommit --jq .mergeCommit.oid)
gh run list --commit "$SHA" --limit 5
```

- Tie the check to the merge commit, not to `--branch main`, which happily returns an
  older run. No run found for that SHA → not verified; say so and stop.
- Once the deploy run has finished green, show the user the list of tickets you are about
  to close and move only the ones they confirm. Skip any ticket that still has an open PR.
- Report the deploy run's result. Never report a release as shipped on the strength of the
  merge alone.
