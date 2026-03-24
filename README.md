# claude-market

Custom plugin marketplace for Claude Code.

## Install

```bash
/plugin marketplace add <your-github-username>/ecb-claude-market
```

## Plugins

| Plugin | Description |
|---|---|
| `plugin-creator` | Scaffolds new plugins interactively — commands, agents, hooks, MCP servers |

## Enable a plugin

```bash
/plugin install plugin-creator@claude-market
```

## Add to team settings

In `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-market": {
      "source": {
        "source": "github",
        "repo": "<your-github-username>/ecb-claude-market"
      }
    }
  },
  "enabledPlugins": {
    "plugin-creator@claude-market": true
  }
}
```
