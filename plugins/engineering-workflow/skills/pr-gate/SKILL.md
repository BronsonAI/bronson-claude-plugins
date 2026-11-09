---
name: pr-gate
description: Review one or more GitHub pull requests against the Bronson review policy and post the verdict to each PR. Use when asked to review, gate, check or sign off a PR, to review all open PRs, or before marking a draft ready. Also invoked by pr-create, fix-review, hotfix and release.
---

# pr-gate

Reviews pull requests and posts a verdict. This skill **is** the Bronson review policy —
the passes and severities below are the policy, versioned here rather than in a separate
`REVIEW.md`.

pr-gate reports. It does not fix. Use `fix-review` to act on findings.

## 1. Resolve the targets

Accept PR numbers, URLs, `ENG-###` keys, or nothing at all.

```bash
gh pr list --repo <owner/repo> --state open --json number,title,author,headRefName,baseRefName,isDraft
```

- Nothing given and the current branch has a PR → that one: `gh pr view --json number`.
- "all my open PRs" → `gh pr list --author @me --state open`.
- An `ENG-###` key → match it against open PR head branch names.
- Several targets → gate each independently, then report one summary table to the user.

## 2. Gather the evidence

Per PR, before forming any opinion:

```bash
gh pr view <n> --json number,title,body,author,headRefName,baseRefName,isDraft,additions,deletions,files,labels
gh pr diff <n>
gh pr checks <n>
```

Then:

- **The Linear ticket.** Pull the `ENG-###` from the branch name or PR body and read the
  issue — it is the scope of record. If the PR references no ticket, that is a finding.
- **The committed artifacts.** Read `docs/tickets/{KEY}/plan.md` and `spec.md` on the
  branch if present. `plan.md` is what the diff gets checked against.
- **Checks still running.** Do not gate on a pending suite. Report the pending checks and
  either wait for them or say plainly that the verdict excludes CI.

Read the actual diff. Never infer what a PR does from its title or body.

## 3. The review passes

Run every pass. Each finding carries a file, a line, and a concrete failure — the inputs
or state that produce the wrong result. A finding you cannot state a failure for is an
opinion; drop it.

**P1 — Blocking.** Any one of these fails the PR on its own.
- Correctness bugs: wrong logic, off-by-one, unhandled null, broken error path, race.
- Data loss or corruption: destructive migration without a snapshot, unguarded delete,
  a backfill with no batching or resume.
- Security: committed secrets or keys, missing authn/authz, injection, SSRF, unsafe
  deserialisation, a dependency added with a known advisory.
- **PHI/PII exposure.** Bronson handles medical records under HIPAA. Flag PHI in logs,
  error messages, analytics payloads, LLM prompts sent to third parties, and fixtures.
- Tenant or case isolation broken: a query that can read across tenants.
- Breaking an published interface without a version bump or migration path.
- A required status check failing.

**P2 — Scope.** The diff against the ticket and the plan.
- Work in the diff that the ticket does not ask for. Name it; do not quietly accept it.
- `plan.md` steps with no corresponding change, or a diff that contradicts the plan with
  no plan update in the same commit.
- Ticket acceptance criteria not met.

**P3 — Convention.** Mechanical, from `bronson-engineering-workflow`.
- Branch name matches `{type}/{first-name}-{KEY}-{description}`.
- Base branch is right: `develop` for feature/fix/improvement, or the parent branch if
  stacked; `main` only for a hotfix or a release.
- Commit messages carry the `[ENG-###]` prefix.
- The artifacts the ticket type requires are present (Feature: intent+spec+plan; Bug:
  intent+plan; Improvement: plan).
- `prod` label present and no downtime line stated in the PR body; `destructive` label
  present and no snapshot step.

**P4 — Quality.** Not blocking, worth saying.
- Changed behaviour with no test covering it.
- Duplication of something that already exists in the repo — name the existing thing.
- Dead code, commented-out blocks, stray debug logging, leftover scratch files.
- Naming or structure that will mislead the next reader.

**P5 — Accessibility.** Only if the diff touches UI.
- Obvious gaps: a control with no accessible name, an image with no alt, a click handler
  on a non-interactive element, a colour-only state change.
- For anything more than obvious, say so and point at `check-accessibility` rather than
  guessing from the diff.

## 4. The verdict

| Verdict | Condition |
|---|---|
| `BLOCK` | Any P1 finding, or a required check failing |
| `CHANGES REQUESTED` | No P1, but one or more P2 or P3 findings |
| `PASS WITH NOTES` | Only P4 or P5 findings |
| `PASS` | Nothing found |

## 5. Post it

**On a PR you authored, post an issue comment.** GitHub rejects `APPROVE` and
`REQUEST_CHANGES` on your own pull request with a 422 — a formal review is not available,
so do not attempt one and do not report the failure as a problem with the PR.

```bash
gh pr comment <n> --body-file <path>
```

**On someone else's PR, post a formal review:**

```bash
gh pr review <n> --request-changes --body-file <path>   # BLOCK or CHANGES REQUESTED
gh pr review <n> --comment --body-file <path>           # PASS WITH NOTES
gh pr review <n> --approve --body-file <path>           # PASS
```

Write the body to a file and use `--body-file`; passing multi-line markdown inline
through a shell mangles it.

**Replace your previous verdict rather than stacking another one.** Begin every body with
the marker line below, then before posting, look for an existing comment carrying it and
edit that one instead.

```
<!-- bronson-pr-gate -->
```

```bash
# find a previous verdict
gh api repos/<owner/repo>/issues/<n>/comments --jq '.[] | select(.body | contains("<!-- bronson-pr-gate -->")) | .id'
# update it in place — -F reads the value from a file when it starts with @
gh api -X PATCH repos/<owner/repo>/issues/comments/<id> -F body=@<path>
```

## 6. Verdict format

```markdown
<!-- bronson-pr-gate -->
## pr-gate: CHANGES REQUESTED

**ENG-115** · `feature/kevin-ENG-115-case-intake-postgres` → `develop` · 14 files, +612/-38
Checks: 2 passing, 0 failing

### Findings

| # | Sev | Where | Finding |
|---|-----|-------|---------|
| 1 | P2 | `apps/api/src/intake/router.ts:88` | Adds a `/intake/export` route the ticket does not mention and `plan.md` does not list. |
| 2 | P3 | — | Commits `a1b2c3d` and `e4f5g6h` are missing the `[ENG-115]` prefix. |
| 3 | P4 | `apps/api/src/intake/service.ts:142` | `normalizeAddress` duplicates `packages/common/src/address.ts`. |

### 1 — Unplanned export route (P2)

`router.ts:88` registers `GET /intake/export`, which streams the full intake table as
CSV. ENG-115 scopes this ticket to the Postgres schema and the write path; nothing in the
ticket or `plan.md` covers an export. Either drop it from this PR or add it to the ticket
and the plan first.

...one section per finding, in severity order...

### What passes
Schema migration is reversible and batched. No PHI in the new log lines. Tests cover the
write path.
```

Keep the prose sections to the findings that need explaining. A P3 convention finding
usually needs only its table row.

## 7. Reporting back to the user

One line per PR, with the link, then the detail for anything blocking:

```
#42  ENG-115  CHANGES REQUESTED  2 scope/convention findings   <url>
#43  ENG-118  PASS WITH NOTES    1 duplication note            <url>
```

State what you actually verified. If CI was still running, or you could not read the
Linear ticket because the connector was not authorised, say so — do not let a missing
input read as a clean pass.
