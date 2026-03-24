---
name: new-plugin
description: Scaffold a new plugin — provide a name and description to get started
---

You are scaffolding a new Claude Code plugin. The user has provided: $ARGUMENTS

Parse the input to extract:
- **name**: plugin name (kebab-case). If not kebab-case, convert it.
- **description**: what the plugin does
- **components**: any mentioned components (commands/agents/hooks/mcp). Default to `commands` only if unspecified.

Then immediately generate the plugin without asking further questions, using sensible defaults:

1. Create `plugins/<name>/plugin.json`
2. For each component type detected:
   - **commands**: create `plugins/<name>/commands/<name>.md` with a focused prompt
   - **agents**: create `plugins/<name>/agents/<name>-agent.md` with a clear persona
   - **hooks**: add to plugin.json only (no separate file needed)
   - **mcpServers**: add to plugin.json only
3. Add the plugin entry to `.claude-plugin/marketplace.json`

After generating, output a file tree of what was created and the install command:
```
/plugin install <name>@ecb-claude-market
```

If the input is missing a name or description, ask for just those two things before proceeding.
