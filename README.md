# ECB Claude Marketplace

Custom plugin marketplace for Claude Code.

## Install

```bash
/plugin marketplace add <your-github-username>/ecb-claude-market
```

## Plugins

| Plugin | Description |
|---|---|
| `code-review` | `/review`, `/explain`, `/security` commands |
| `git-workflow` | `/commit`, `/pr`, `/changelog` + git-agent + post-edit hooks |
| `mcp-tools` | MCP servers: filesystem, web fetch, SQLite |

## Enable a plugin

```bash
/plugin install code-review@ecb-claude-market
/plugin install git-workflow@ecb-claude-market
/plugin install mcp-tools@ecb-claude-market
```

## Add to team settings

In `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "ecb-claude-market": {
      "source": {
        "source": "github",
        "repo": "<your-github-username>/ecb-claude-market"
      }
    }
  },
  "enabledPlugins": {
    "code-review@ecb-claude-market": true,
    "git-workflow@ecb-claude-market": true,
    "mcp-tools@ecb-claude-market": true
  }
}
```
