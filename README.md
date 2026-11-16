# bronson-claude-plugins

The internal Claude Code plugin marketplace for Bronson AI engineering. It is deployed
org-wide through managed settings, so everything here is live on every seat without
anyone installing anything.

## The skills

| Skill | What it does |
|---|---|
| `bronson-engineering-workflow` | The conventions the rest build on — Linear, branch naming, the develop/main strategy, commits, artifacts, docs placement, toolchain |
| `start-ticket` | Reads a Linear ticket, cuts the branch, writes intent/spec/plan, stops at the plan-approval gate |
| `pr-create` | Gates the branch and fixes what blocks, opens a draft PR, then marks it ready and requests `BronsonAI/engineering` |
| `pr-gate` | Reviews a PR, or a branch before its PR exists, against the review policy. Preflight on your own PR; asks before reviewing someone else's |
| `fix-review` | Triages review feedback, applies it, replies to each thread, re-gates, re-requests the reviewer |
| `hotfix` | Branches from `main`, ships the fix, opens the back-merge PR down to `develop` |
| `release` | Builds the changelog and opens the `develop` to `main` promotion PR |
| `check-accessibility` | WCAG 2.2 AA audit across automated, static and manual passes, then fixes what is safe to fix |

They call each other: `start-ticket` leads into `pr-create`, which runs `pr-gate`, whose
findings `fix-review` acts on. `pr-gate` is also the review policy itself — the passes and
severities in that skill are the policy, versioned there rather than in a separate
`REVIEW.md`.

## The workflow they encode

```
Linear ticket
   │  start-ticket      branch + docs/tickets/{KEY}/{intent,spec,plan}.md
   ▼                    ── human gate: plan approved ──
{type}/{first-name}-{KEY}-{description}
   │  pr-create         pr-gate ▸ draft PR ▸ checks ▸ ready ▸ request engineering
   │  fix-review        feedback ▸ fixes ▸ replies ▸ re-gate
   ▼
develop ──▶ dev env
   │  release           changelog ▸ promotion PR ── human gate: deploy authorised ──
   ▼
main ──▶ prod env
   ▲  hotfix            main ▸ fix ▸ PR to main ▸ back-merge PR to develop
```

**Two branching models.** The diagram above is the `develop` → `main` flow, which applies
to every actively developed repo unless it is explicitly exempt. Exempt today:
`infra-live` and `infra-modules`, which are `main`-only — there, branches base on the
default branch, PRs target it, and `release` and the `hotfix` back-merge do not apply. The
skills detect which model a repo uses (`git ls-remote --heads origin develop`) rather than
assuming; none of them creates a `develop` branch to make a repo fit. A repo's own
`CLAUDE.md` wins over all of this, including on what a merge deploys.

Artifacts are committed to the branch rather than kept in Linear, so a reviewer reads
intent, spec and plan in the PR diff and the trail survives in git history for audit. The
Linear ticket stays authoritative for *whether and what*; the committed files are
authoritative for *how*. Ceremony scales to the ticket type — a `Feature` gets all three
files, a `Bug` gets intent and plan, an `Improvement` gets a plan.

Based on Anthropic's [AI-native SDLC playbook](https://claude.com/resources/articles/the-ai-native-sdlc-playbook).

## Layout

```
.claude-plugin/marketplace.json          the marketplace catalog
plugins/engineering-workflow/
  .claude-plugin/plugin.json             manifest + bundled MCP servers
  skills/<skill-name>/SKILL.md           one directory per skill
ADMIN-SETUP.md                           how the org-wide rollout is configured
```

## Changing a skill

`plugin.json` carries no `version` field on purpose: every commit to `main` is a new
version. Seats pick it up automatically only while `autoUpdate: true` is set on the
managed marketplace entry (see `ADMIN-SETUP.md`); otherwise a seat stays on the version it
last pulled until someone runs `/plugin marketplace update bronson-tools`.
That cuts both ways — **a merge to `main` changes how Claude behaves for everyone**, so
changes go through a PR like any other code.

Adding a *skill* needs nothing from an admin, because managed settings enable the
`engineering-workflow` plugin as a whole. Adding a new *plugin* would need the admin
settings updated, which is why these all live in one plugin.

Validate before merging:

```bash
claude plugin validate .
```

## MCP servers

The plugin bundles Linear and Notion over HTTP, each authenticating by browser OAuth on
first use. The skills themselves drive GitHub through the `gh` CLI, not MCP.
`strictPluginOnlyCustomization: ["mcp"]` in managed settings stops *user and project* MCP
servers — including a repo's `.mcp.json` — but not claude.ai connectors. See
`ADMIN-SETUP.md`.
