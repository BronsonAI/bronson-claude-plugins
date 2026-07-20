# bronson-claude-plugins

Internal Claude Code plugin marketplace for Bronson AI engineering.

## What's in here

```
.claude-plugin/marketplace.json          — the marketplace catalog
plugins/engineering-workflow/
  .claude-plugin/plugin.json             — plugin manifest + bundled MCP servers
  skills/bronson-engineering-workflow/
    SKILL.md                             — the workflow instructions
ADMIN-SETUP.md                           — org-wide rollout steps (read this next)
```

## Before you push this

Fill in the `<!-- TODO -->` placeholders in `SKILL.md`:

1. Your Linear team name/key (if you use one)
2. Branch naming convention
3. Which Notion space new docs should live under

## Quick start (test locally first)

```bash
git init
git add .
git commit -m "Initial engineering-workflow plugin"
gh repo create bronson-ai/claude-plugins --private --source=. --push
```

Then, from any Claude Code session:

```
/plugin marketplace add bronson-ai/claude-plugins
/plugin install engineering-workflow@bronson-tools
/mcp
```

Confirm `linear`, `github`, and `notion` show up and that each authenticates via browser
OAuth on first use. Once you're happy with it, follow `ADMIN-SETUP.md` to make it the
default for everyone.

## Validate before distributing

```bash
claude plugin validate .
```

Run this from the repo root to catch marketplace/plugin JSON errors before anyone else
pulls it.
