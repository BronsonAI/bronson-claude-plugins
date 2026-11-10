# Admin setup — deploy org-wide in Claude Code

This repo is a plugin marketplace. An Enterprise Owner enables it for everyone by pasting
the block below into managed settings (Claude admin console → server-managed settings, or
managed-settings.json / MDM profile).

The repo path below is this repository. It is already deployed org-wide — this doc is the
record of how, and what to change if it ever moves.

```json
{
  "extraKnownMarketplaces": {
    "bronson-tools": {
      "source": { "source": "github", "repo": "BronsonAI/bronson-claude-plugins" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "engineering-workflow@bronson-tools": true
  },
  "strictKnownMarketplaces": [
    { "source": "github", "repo": "BronsonAI/bronson-claude-plugins" }
  ],
  "strictPluginOnlyCustomization": ["mcp"]
}
```

What each key does:

- `extraKnownMarketplaces` — auto-registers this marketplace for every seat. No one runs
  `/plugin marketplace add` manually.
- `enabledPlugins` — turns the plugin on by default. No one runs `/plugin install` manually.
- `strictKnownMarketplaces` — locks marketplace sources to this repo only, so no one can add
  a rogue or stale substitute.
- `strictPluginOnlyCustomization: ["mcp"]` — MCP servers can only come from plugins (i.e.
  this one). This blocks *user and project* MCP servers, including a repo's own
  `.mcp.json`. It does **not** block claude.ai connectors, which load separately and can
  still provide a personal Linear, Notion or Drive connection alongside the plugin's
  servers; set `disableClaudeAiConnectors: true` as well if that is the intent.
- `autoUpdate: true` — without it, background plugin updates are off and seats keep
  whatever version they last pulled.

## Verify it landed

Have a teammate run `/status` inside Claude Code. It should report an
`Enterprise managed settings` line with a source in parentheses — `(remote)` if delivered
from the admin console. Then run `/mcp` — they should see `linear` and `notion`
listed, and get a browser OAuth prompt the first time they use each one. There is no
`github` server: the skills drive GitHub through the `gh` CLI, which authenticates on its
own.

## Repo-level fallback (optional, for unmanaged devices)

For contractor laptops or any device not covered by managed settings, commit this to
`.claude/settings.json` in your actual engineering repos (e.g. FOIAger):

```json
{
  "extraKnownMarketplaces": {
    "bronson-tools": {
      "source": { "source": "github", "repo": "BronsonAI/bronson-claude-plugins" }
    }
  },
  "enabledPlugins": {
    "engineering-workflow@bronson-tools": true
  }
}
```

This prompts anyone who trusts the project folder to install the marketplace, even without
admin policy.

## Updating the skill later

`plugin.json` has no `version` field on purpose — every commit to `main` counts as a new
version. Seats pick it up only when the marketplace updates: that needs `autoUpdate: true`
on the managed `bronson-tools` entry above. Without it, each seat must run
`/plugin marketplace update bronson-tools` by hand, and seats drift apart silently.

Add new toolchain-decision entries to `skills/bronson-engineering-workflow/SKILL.md` §7
**through a PR** — never pushed straight to `main`. A merge here changes Claude's
behaviour for everyone, which is exactly why it gets reviewed.
