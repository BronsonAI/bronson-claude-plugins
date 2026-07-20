# Admin setup — deploy org-wide in Claude Code

This repo is a plugin marketplace. Once it's pushed to GitHub, an Enterprise Owner
enables it for everyone by pasting the block below into managed settings (Claude admin
console → server-managed settings, or your managed-settings.json / MDM profile).

Replace `bronson-ai/claude-plugins` with your actual GitHub repo path.

```json
{
  "extraKnownMarketplaces": {
    "bronson-tools": {
      "source": { "source": "github", "repo": "bronson-ai/claude-plugins" }
    }
  },
  "enabledPlugins": {
    "engineering-workflow@bronson-tools": true
  },
  "strictKnownMarketplaces": [
    { "source": "github", "repo": "bronson-ai/claude-plugins" }
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
  this one), so no one quietly swaps in a personal Notion/GitHub/Linear connection instead.

## Verify it landed

Have a teammate run `/status` inside Claude Code. It should report an
`Enterprise managed settings` line with a source in parentheses — `(remote)` if delivered
from the admin console. Then run `/mcp` — they should see `linear`, `github`, and `notion`
listed, and get a browser OAuth prompt the first time they use each one.

## Repo-level fallback (optional, for unmanaged devices)

For contractor laptops or any device not covered by managed settings, commit this to
`.claude/settings.json` in your actual engineering repos (e.g. FOIAger):

```json
{
  "extraKnownMarketplaces": {
    "bronson-tools": {
      "source": { "source": "github", "repo": "bronson-ai/claude-plugins" }
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
version, and users pick it up automatically (no re-release step, no admin action needed).
Add new toolchain-decision entries directly to `SKILL.md` and push.
