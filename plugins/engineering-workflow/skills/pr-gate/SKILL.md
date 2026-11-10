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

Then sort every finding you keep into one of two buckets. Sort by consequence, not by
category: the question is always "if this merges as is, what goes wrong?"

### Blocking — the PR cannot merge

- Correctness bugs: wrong logic, off-by-one, unhandled null, broken error path, race.
- Data loss or corruption: destructive migration without a snapshot, unguarded delete,
  a backfill with no batching or resume.
- Security: committed secrets or keys, missing authn/authz, injection, SSRF, unsafe
  deserialisation, a dependency added with a known advisory.
- **PHI/PII exposure.** Bronson handles medical records under HIPAA. Flag PHI in logs,
  error messages, analytics payloads, LLM prompts sent to third parties, and fixtures.
- Tenant or case isolation broken: a query that can read across tenants.
- Breaking a published interface without a version bump or migration path.
- A required status check failing.
- Wrong base branch, or `destructive` label with no snapshot step.
- Work the ticket does not ask for that carries its own risk — a second migration, a new
  endpoint, a dependency bump. Name it.

### Non-blocking — say it, approve anyway

- Convention: branch name, `[ENG-###]` commit prefixes, missing ticket, missing artifacts.
- Changed behaviour with no test covering it.
- Duplication of something that already exists — name the existing thing.
- Dead code, commented-out blocks, stray debug logging, leftover scratch files.
- Naming or structure that will mislead the next reader.
- Scope creep that is merely untracked, not risky.
- Accessibility, where the diff touches UI: a control with no accessible name, an image
  with no alt, a click handler on a non-interactive element, a colour-only state change.
  For anything beyond the obvious, point at `check-accessibility` rather than guessing.

Before you file a convention finding, check whether the repo follows that convention at
all. A gap the whole repo shares is the repo's, not this author's — say so in the one
line, or drop it.

## 4. The verdict

| Verdict | Condition |
|---|---|
| `CHANGES REQUESTED` | One or more blocking findings |
| `APPROVED` | No blocking findings — whatever the non-blocking count |

## 5. Post it

**On a PR you authored, post an issue comment.** GitHub rejects `APPROVE` and
`REQUEST_CHANGES` on your own pull request with a 422 — a formal review is not available,
so do not attempt one and do not report the failure as a problem with the PR.

```bash
gh pr comment <n> --body-file <path>
```

**On someone else's PR, post a formal review:**

```bash
gh pr review <n> --request-changes --body-file <path>   # CHANGES REQUESTED
gh pr review <n> --approve --body-file <path>           # APPROVED
```

Write the body to a file and use `--body-file`; passing multi-line markdown inline
through a shell mangles it.

Begin every body with the marker line `<!-- bronson-pr-gate -->`. On a PR you authored,
look for an existing comment carrying it and edit that one in place rather than stacking
a second verdict:

```bash
gh api repos/<owner/repo>/issues/<n>/comments --jq '.[] | select(.body | contains("<!-- bronson-pr-gate -->")) | .id'
gh api -X PATCH repos/<owner/repo>/issues/comments/<id> -F body=@<path>
```

## 6. Verdict format

Findings only. No tables — a four-column table squeezes its headers onto three lines and
is unreadable. No "what passes" section, no summary of the diff, no restating the PR
body. The author knows what they wrote; tell them only what to change.

A blocking finding gets a heading and at most five sentences: where it is, the inputs
that break it, what goes wrong. A non-blocking finding gets exactly one line. If a
blocking finding needs more than five sentences, the extra belongs in a code comment on
the line, not the verdict.

```markdown
<!-- bronson-pr-gate -->
## pr-gate: CHANGES REQUESTED

**ENG-115** · `feature/kevin-ENG-115-case-intake-postgres` → `develop` · 14 files, +612/-38
Checks: 2 passing, 0 failing

### Blocking

**`apps/api/src/intake/service.ts:142` — the retry drops the last batch.**
`flush()` is called inside the loop but never after it, so a run whose row count is not a
multiple of `BATCH` leaves the remainder unwritten. A 2,500-row intake with `BATCH=1000`
writes 2,000 rows and reports success.

### Non-blocking

- `apps/api/src/intake/router.ts:88` — `/intake/export` is not in ENG-115 or `plan.md`.
- `apps/api/src/intake/service.ts:96` — `normalizeAddress` duplicates `packages/common/src/address.ts`.
- Commits `a1b2c3d` and `e4f5g6h` have no `[ENG-115]` prefix (most of this repo's do not either).
```

Approving with notes is the same body with `## pr-gate: APPROVED` and no Blocking
section. Approving clean is two lines: the header and `No findings.`

**Re-gating after fixes.** Verify each fix against the diff, then write the verdict as if
it were the first one: only what still needs changing. Never a "fixed" or "verified"
section, never a diff of your own previous verdict — nobody reads it. If nothing blocking
remains, approve and list whatever non-blocking items are left. Say what you could not
verify, and nothing else about the past.

## 7. Reporting back to the user

One line per PR, then the blocking findings in one sentence each. Nothing else — no
recap of what was fixed, no restating the verdict body, no summary of your process.

```
#42  ENG-115  CHANGES REQUESTED  1 blocking, 3 notes   <url>
#43  ENG-118  APPROVED           2 notes               <url>
```

State what you did not verify: CI still running, a ticket you could not read, a path you
did not exercise. A missing input must never read as a clean pass.
