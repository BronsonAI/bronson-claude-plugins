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
- Nothing given and the current branch has no PR → show the open PR list and ask which to
  gate. Never gate every open PR without an explicit "all" request.
- "all my open PRs" → `gh pr list --author @me --state open`.
- An `ENG-###` key → match it against open PR head branch names.
- Several targets → gate each independently, then report one summary table to the user.
- **A branch with no PR yet**, when `pr-create` calls the gate before creating one → gate
  the branch. Everything below works the same; only the evidence commands change.

## 2. Gather the evidence

Per PR, before forming any opinion:

```bash
gh pr view <n> --json number,title,body,author,headRefName,baseRefName,isDraft,additions,deletions,files,labels
gh pr diff <n>
gh pr checks <n>
```

On a branch with no PR, read the same diff from git and take the ticket from the branch
name and the commits instead of a PR body:

```bash
git fetch origin
git diff origin/<base>...HEAD
git log --oneline origin/<base>..HEAD
```

Required checks cannot run before the PR exists. Say so, run every other pass, and leave
the check verdict to the gate that runs after the draft is opened.

Then:

- **The Linear ticket.** Pull the `ENG-###` from the branch name or PR body and read the
  issue — it is the scope of record. If the PR references no ticket, that is a finding.
- **The committed artifacts.** Read `docs/tickets/{KEY}/plan.md` and `spec.md` on the
  branch if present. `plan.md` is what the diff gets checked against.
- **Checks still running.** `gh pr checks <n> --required --watch`. Read the exit code
  carefully, because three different outcomes are non-zero:
  - **no checks reported** — ambiguous, and the two cases are not the same. Ask the base
    branch whether it *has* required checks:

    ```bash
    gh api repos/<owner/repo>/branches/<base>/protection --jq '.required_status_checks.contexts'
    ```

    None configured → genuinely nothing to wait for. Say "no required checks" and move
    on. Some configured but none reported → they have not registered yet, which is normal
    straight after `gh pr create`. Wait and ask again; never mark a PR ready on it.
  - **8** — still pending (without `--watch`; with `--watch` the command waits instead).
  - **a check actually failed** — blocking.

  Never report any of the first two as a failing check. Only a *required* check counts as
  blocking — an optional job going red is a note.

Read the actual diff. Never infer what a PR does from its title or body.

**Modes.** Three kinds of PR are not feature PRs, and the convention checks do not apply:

| PR | How to tell | What to run |
|---|---|---|
| Release | head `develop`, base `main` | Blocking pass only, plus the migration and config flags and whether `main` is behind. Skip branch name, ticket, artifacts, commit prefixes — merge commits and unkeyed commits are expected. |
| Back-merge | head `backmerge/*`, base `develop` | Blocking pass only. Base `develop` is correct here. Never rename the branch or rewrite its commits: they are shared with the hotfix PR and with `main`. |
| Hotfix | head `hotfix/*`, base `main` | Base `main` is correct — that is the whole point of a hotfix. Run every other pass; the ticket and the `prod` downtime line matter more here, not less. |
| Exempt repo | no `develop` branch | Base is the default branch, and that is correct. Skip the base-branch check entirely. |

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
- Wrong base branch for the repo's flow (see Modes).
- A `prod` ticket whose PR body states no downtime line, or a `destructive` ticket whose
  body states no snapshot step. `prod` and `destructive` are **Linear** labels, read from
  the ticket in step 2 — not GitHub PR labels, which nothing sets. If the ticket could not
  be read, say the check is unverified rather than passing it.

  This one needs the PR body, so in branch mode (no PR yet) check the body text
  `pr-create` is about to use, and say which you checked. It is the one blocking finding
  that a pre-creation gate reads from the draft body rather than the diff.
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

## 5. Your own PR: report, do not post

**On a PR you authored the gate is a preflight check, not a review.** Print the verdict
in the terminal and stop. Do not comment on the PR.

The value is in the author seeing it before a reviewer spends time: the missing ticket,
the wrong base, the failing check. None of that value reaches the PR thread. What does
reach it is noise above the real review and a green stamp that invites skimming — this
skill's own PR carried a self-posted `APPROVED` while a reviewer was finding thirty-five
issues in it. GitHub refusing to let you approve your own pull request is the platform
making the same point.

If the user explicitly asks for the verdict on the PR, post it with
`gh pr comment <n> --body-file <path>` — but that is their call, never the default.

## 5b. Someone else's PR: confirm, then post

**Show the verdict and get an explicit yes before submitting it.** `--approve` and
`--request-changes` are real reviews under the user's name: an approval can satisfy a
branch protection rule and clear the way to a merge, and a request for changes blocks a
teammate. Neither is yours to submit on a reading of a diff. Print the verdict body, say
which of the two you are about to post, and wait. "Review this PR" asks for the review,
not for it to be filed.

If the user does not want to post, give them the body and stop. `--comment` carries no
state and is the middle ground when they are unsure.

Once they have said yes:

```bash
gh pr review <n> --request-changes --body-file <path>   # CHANGES REQUESTED
gh pr review <n> --approve --body-file <path>           # APPROVED
```

Write the body to a file and use `--body-file`; passing multi-line markdown inline
through a shell mangles it.

A review's state cannot be changed once submitted, so a re-gate posts a *new* review
rather than replacing the old one. Submitting the new verdict is what changes the PR's
status; the earlier review stays in the timeline and that is expected.

## 6. Verdict format

Findings only. No tables — a four-column table squeezes its headers onto three lines and
is unreadable. No "what passes" section, no summary of the diff, no restating the PR
body. The author knows what they wrote; tell them only what to change.

A blocking finding gets a heading and at most five sentences: where it is, the inputs
that break it, what goes wrong. A non-blocking finding gets exactly one line. If a
blocking finding needs more than five sentences, the extra belongs in a code comment on
the line, not the verdict.

```markdown
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
