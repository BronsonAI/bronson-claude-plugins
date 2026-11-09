# bronson-claude-plugins

The internal Claude Code plugin marketplace for Bronson AI engineering. It is deployed
org-wide through managed settings, so everything here is live on every seat without
anyone installing anything.

## The skills

| Skill | What it does |
|---|---|
| `bronson-engineering-workflow` | The conventions the rest build on — Linear, branch naming, the develop/main strategy, commits, artifacts, docs placement, toolchain |
| `start-ticket` | Reads a Linear ticket, cuts the branch, writes intent/spec/plan, stops at the plan-approval gate |
| `pr-create` | Opens a draft PR, gates it, then marks it ready and requests `BronsonAI/engineering` |
| `pr-gate` | Reviews one or more PRs against the review policy and posts the verdict |
| `fix-review` | Applies review feedback, replies to each thread, re-gates |
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
feature/{first-name}-{KEY}-{description}
   │  pr-create         draft PR ▸ pr-gate ▸ ready ▸ request engineering
   │  fix-review        feedback ▸ fixes ▸ replies ▸ re-gate
   ▼
develop ──▶ dev env
   │  release           changelog ▸ promotion PR ── human gate: deploy authorised ──
   ▼
main ──▶ prod env
   ▲  hotfix            main ▸ fix ▸ PR to main ▸ back-merge PR to develop
```

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
version and seats pick it up automatically. There is no release step and no admin action.
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

The plugin bundles Linear, GitHub and Notion over HTTP, each authenticating by browser
OAuth on first use. `strictPluginOnlyCustomization: ["mcp"]` in managed settings means MCP
servers can only come from this plugin, so nobody quietly substitutes a personal
connection. See `ADMIN-SETUP.md`.
