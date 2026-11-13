---
name: bronson-engineering-workflow
description: Bronson AI engineering conventions — Linear ticketing, branch naming, the develop/main branching strategy, commit and PR rules, where SDLC artifacts and documentation live, and the default toolchain. Use when starting engineering work, naming a branch, deciding a PR's base branch, asking where a doc belongs, or whenever another Bronson skill refers to "the conventions".
---

# Bronson AI — Engineering Workflow

The conventions every other Bronson skill builds on. `start-ticket`, `pr-create`,
`pr-gate`, `fix-review`, `hotfix` and `release` each restate the rules they need, so you
do not have to load this first — read it when you need the whole picture.

## 1. Linear is the scope of record

Work is tracked on the **Engineering** team (key `ENG`) in the Bronson AI Linear
workspace. The ticket defines scope. If a request conflicts with or grows beyond the
ticket, say so rather than silently widening the work.

- Statuses: `Backlog` → `Needs Triaged` → `Todo` → `In Progress` → `In Review` → `Done`,
  with `Blocked`, `Canceled` and `Duplicate` off to the side.
- The type labels are `Bug`, `Feature` and `Improvement`. One of them belongs on every
  ticket; it drives the branch type and which artifacts get written.
- Two labels carry hard rules, stated in Linear itself:
  - `prod` — touches production. Needs its own reviewed PR and a stated downtime line
    before merge.
  - `destructive` — deletes resources. Needs its own reviewed PR, and snapshots first.
- Never create a ticket without confirming the content with the user first. Draft it,
  show it, then create it.

## 2. Branch naming

```
{type}/{first-name}-{KEY}-{description}
```

- `type` — `feature`, `fix`, `chore` or `hotfix`. `feature` is new functionality, `fix`
  corrects a defect, `chore` is tooling, dependencies, CI and docs. A `Feature` ticket
  is `feature` and a `Bug` is `fix`; an `Improvement` ticket takes whichever of
  `feature`, `fix` or `chore` fits the change. `hotfix` overrides the label: a `Bug`
  shipped through `hotfix` is `hotfix/`, not `fix/`. Work that will never have a ticket
  is `chore` and drops the `{KEY}` segment. That is different from work whose ticket
  does not exist *yet*: a hotfix shipped before its ticket is
  confirmed uses `NOTICKET-<yyyymmddhhmm>` in the key's place and keeps it. Once the key
  exists it goes in the PR titles and bodies; the branch is never renamed (`hotfix` step 2).
- `first-name` — the author's first name, lowercase. Not a username.
- `KEY` — the Linear key, uppercase, matching the commit prefix and the artifact
  directory so all three read the same.
- `description` — kebab-case, five words at most.

```
feature/kevin-ENG-115-case-intake-postgres
fix/suraj-ENG-207-duplicate-webhook-retry
hotfix/kevin-ENG-221-token-refresh-loop
chore/kevin-bump-node-22
```

Two names are reserved and sit outside the format: `backmerge/main-to-develop-{KEY}`
(`hotfix` step 7) and `develop` itself as the head of a release PR.

This is the standard for new branches, and it is deliberately not what older branches in
the org look like. Do not rename a branch that is already pushed to make it match — the
cost of broken links outweighs the tidiness. A repo's own `CLAUDE.md` or `CONTRIBUTING`
wins over this section.

## 3. Branching strategy

```
feature/* ──▶ develop ──▶ main
                 │          │
              dev env    prod env
```

**Which repos this applies to.** Every actively developed repo, unless it is explicitly
exempt. Exempt today: `infra-live` and `infra-modules`, which branch off `main` — say
"exempt", not "there is no develop". Dormant and archived repos are out of scope.

Check once, at the start of any skill that bases or targets a branch:

```bash
git ls-remote --exit-code --heads origin develop >/dev/null 2>&1 && echo develop || echo exempt
```

In an exempt repo the integration branch is the default branch
(`gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`): base there, PR
there, skip `release` and the `hotfix` back-merge, and do not claim what a merge deploys
— read the repo's `CLAUDE.md` for that. Never create a `develop` branch to make a repo
fit this section.

- Feature, fix and chore branches base on **`develop`**. CI/CD deploys `develop`
  to the dev environment.
- Promotion is by merging the whole branch upward: `develop` into `main`, which CI/CDs
  to production. Use `release` for that PR. Cherry-picking is an exception that needs a
  stated reason, not a routine shortcut.
- A `staging` branch exists between them in some repos and is **not in use today**.
  Treat `develop` to `main` as the only promotion path until that changes.
- Hotfixes are the one upward exception: branch from `main`, PR into `main`, then open a
  back-merge PR from `main` down to `develop` so `develop` has the fix too. Use
  `hotfix`.
- Never commit directly to `main` or `develop`. Every change goes through a PR, however
  small.

### Stack PRs rather than branching off develop each time

If you already have an open PR in the repo, branch the next one off **that PR's head**
and pass `--base <that-branch>`, instead of cutting again from `develop`.

The reason is concrete: successive PRs in a repo tend to touch the same few shared files
— `package.json`, lockfiles, CI config — so two branches cut from `develop` conflict the
moment one of them merges. Stacking serialises that.

When a parent is merged through GitHub, GitHub retargets its open children to the
parent's base, including when the merge deletes the branch. What breaks that is a parent
branch deleted **outside** that flow, or a parent PR **closed unmerged** and then tidied
up: the child PR closes, and GitHub refuses both to reopen it and to change its base.

So: never delete a parent branch by hand, and before merging a child confirm its base is
`develop`. Otherwise leave the base alone and let GitHub retarget on merge — retargeting
early puts the parent's unmerged commits in the child's diff, so reviewers and the gate
read the parent's code under the child's ticket.

Retarget by hand in exactly two cases: a parent you are about to close unmerged, and a
parent branch somebody other than GitHub is about to delete.

## 4. Commits

Prefix with the Linear key in brackets:

```
[ENG-115] Add case intake Postgres schema
```

No rule governs how many commits a PR gets. Split them where it helps a reviewer and
leave them alone where it does not.

## 5. SDLC artifacts

Bronson follows the AI-native SDLC artifact chain, with the artifacts **committed to the
branch** under `docs/tickets/{KEY}/` so a reviewer reads them in the PR diff rather than
switching to Linear, and so the trail survives in git history for audit.

| File | Holds |
|---|---|
| `intent.md` | The problem, who it affects, the outcome wanted, constraints, open questions |
| `spec.md` | The design: behaviour, interfaces, data, failure modes, security and UX notes |
| `plan.md` | Implementation plan — files to touch, order of work, risks, test strategy |

**Scale the ceremony to the work.** Three markdown files on a two-line fix is waste:

| Ticket type | Artifacts |
|---|---|
| `Feature` | `intent.md`, `spec.md`, `plan.md` |
| `Bug` | `intent.md` (reproduction + root cause) and `plan.md`. Add `spec.md` only if the fix changes observable behaviour. |
| `Improvement` | `plan.md` |
| Hotfix | `intent.md` (incident + root cause) and `plan.md`, written alongside the fix — speed first, but do not skip them. |

**When the files and the ticket disagree, the ticket wins on scope.** The Linear ticket
is authoritative for *whether and what*; the committed artifacts are authoritative for
*how*. `start-ticket` seeds `intent.md` from the ticket description. Nothing edits the
ticket's scope from a file without the user saying so.

Deviating from `plan.md` mid-build is fine and expected — update `plan.md` in the same
commit that deviates, so the final diff and the plan still describe the same work.

## 6. Where documentation belongs

- **Repo markdown** — anything scoped to the code: folder READMEs, setup steps, how a
  module works, CONTRIBUTING notes. Plus the `docs/tickets/{KEY}/` artifacts above.
- **Notion** — cross-repo and cross-cutting decisions: architecture, vendor and
  toolchain evaluations, runbooks, anything a teammate needs to find without already
  knowing which repo to look in. Docs live in the Bronson AI team space, under the
  subfolder matching the repository; create the subfolder if it does not exist.

The dividing line: `spec.md` explains the design *of this ticket's change* and lives in
the repo. Notion holds decisions that outlive any one ticket.

## 7. Toolchain

Connected by default through this plugin. Do not propose alternatives unless the user
asks to evaluate a change.

- **Linear** — tickets and scope of record
- **GitHub** — code, PRs, reviews. Use the `gh` CLI for PR work; it is faster and more
  predictable than the MCP server for diffs, checks and review posting.
- **Notion** — documentation and decision records
- **Opik** — LLM evaluation and regression testing for FOIAger; a pre-deployment pipeline
  gated on metric floors and baseline comparisons

## 8. Environment notes that bite

- **A single-branch clone never gets `origin/main` from `git fetch origin`,** and a
  shallow one has no merge base. Its fetch refspec covers only the branch it was cloned
  with, so `origin/main...origin/develop` fails with "unknown revision" however often
  you fetch — and once the branch is there, a shallow clone fails again with "no merge
  base". If `git branch -r` lacks a branch a skill needs, fetch it by name
  (`git fetch origin +<b>:refs/remotes/origin/<b>`), and if
  `git rev-parse --is-shallow-repository` says `true`, add `--unshallow`. `release` and
  `hotfix` always do both for `main` and `develop`. Do not fall back to the compare API
  for the file list: it stops at 300 files without saying so.
- **Node may not be on the shell PATH.** nvm4w installs to `C:\nvm4w\nodejs`; export it
  before any `node`, `npm` or `npx` call if the shell cannot find them.
- **npm 11 does not run dependency install scripts unless approved.** It prints a single
  `npm warn install-scripts` line that is easy to miss, and esbuild's postinstall is what
  fetches its platform binary — so Vite fails later with an unrelated-looking error. After
  installing, run `npm install-scripts ls`, show the user the list, and approve only the
  packages they name. Never `--all`: approving an install script runs arbitrary code from
  a dependency, which is the user's call, not Claude's. Say in the PR that
  `package.json` changed.
- **Local Playwright runs prove less than they look.** `reuseExistingServer` is on outside
  CI, so a suite can attach to an unrelated dev server already on the port and produce
  both false failures and false passes. Before trusting a local run, stop the dev stack
  and run it with `CI=true`, which disables reuse and starts exactly what the config
  declares.
