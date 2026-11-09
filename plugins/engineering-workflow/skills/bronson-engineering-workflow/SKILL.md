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

- `type` — `feature`, `fix`, `improvement` or `hotfix`, from the ticket's type label
  (`Bug` becomes `fix`).
- `first-name` — the author's first name, lowercase. Not a username.
- `KEY` — the Linear key, uppercase, matching the commit prefix and the artifact
  directory so all three read the same.
- `description` — kebab-case, five words at most.

```
feature/kevin-ENG-115-case-intake-postgres
fix/suraj-ENG-207-duplicate-webhook-retry
hotfix/kevin-ENG-221-token-refresh-loop
```

## 3. Branching strategy

```
feature/* ──▶ develop ──▶ main
                 │          │
              dev env    prod env
```

- Feature, fix and improvement branches base on **`develop`**. CI/CD deploys `develop`
  to the dev environment.
- Promotion is by merging the whole branch upward: `develop` into `main`, which CI/CDs
  to production. Use `release` for that PR. Cherry-picking is an exception that needs a
  stated reason, not a routine shortcut.
- A `staging` branch exists between them in some repos and is **not in use today**.
  Treat `develop` to `main` as the only promotion path until that changes.
- Hotfixes are the one upward exception: branch from `main`, PR into `main`, then open a
  back-merge PR from `main` down to `develop` so the fix is not lost on the next
  promotion. Use `hotfix`.
- Never commit directly to `main` or `develop`. Every change goes through a PR, however
  small.

### Stack PRs rather than branching off develop each time

If you already have an open PR in the repo, branch the next one off **that PR's head**
and pass `--base <that-branch>`, instead of cutting again from `develop`.

The reason is concrete: successive PRs in a repo tend to touch the same few shared files
— `package.json`, lockfiles, CI config — so two branches cut from `develop` conflict the
moment one of them merges. Stacking serialises that.

GitHub retargets a child to `develop` automatically when the parent merges, but **only
while the child is open**. Merging a parent *and deleting its branch* closes the child
PR, and GitHub then refuses both to reopen it and to change its base. Retarget the child
to `develop` before merging the parent, or keep the parent's branch until you have.

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

- **Node may not be on the shell PATH.** nvm4w installs to `C:\nvm4w\nodejs`; export it
  before any `node`, `npm` or `npx` call if the shell cannot find them.
- **npm 11 does not run dependency install scripts unless approved.** It prints a single
  `npm warn install-scripts` line that is easy to miss, and esbuild's postinstall is what
  fetches its platform binary — so Vite fails later with an unrelated-looking error. After
  installing, run `npm install-scripts ls` and approve the legitimate ones.
- **Local Playwright runs prove less than they look.** `reuseExistingServer` is on outside
  CI, so a suite can attach to an unrelated dev server already on the port and produce
  both false failures and false passes. Before trusting a local run, stop the dev stack
  and run it with `CI=true`, which disables reuse and starts exactly what the config
  declares.
