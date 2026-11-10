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
# merged hotfixes into main
gh pr list --base main --state merged --limit 50 \
  --json number,headRefName --jq '.[] | select(.headRefName | startswith("hotfix/")) | .headRefName'

# for each key above, look the back-merge up directly rather than listing and filtering
gh pr list --base develop --state merged --search "ENG-221" --json number,headRefName
```

  A merged `hotfix/*` into `main` with no merged `backmerge/*` carrying the same
  `ENG-###` is the thing to stop for. Everything else is normal promotion history.

  Look each key up rather than listing back-merges and filtering: a `--limit` applies
  before the jq filter, and on `develop` the recent merges are mostly feature PRs, so a
  back-merge older than the window drops out and its hotfix looks outstanding — stopping
  the release for nothing. Merging `develop` into `main` is a three-way merge and does not
  by itself revert a `main`-only commit.

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
