---
name: new-command
description: Add a new slash command to an existing plugin
---

Add a new command to an existing plugin. Input: $ARGUMENTS

Expected format: `<plugin-name> <command-name> <description>`

Steps:
1. Read `plugins/<plugin-name>/plugin.json` to verify the plugin exists
2. Create `plugins/<plugin-name>/commands/<command-name>.md` with:
   - YAML frontmatter: `name` and `description`
   - A focused, actionable system prompt for the command
   - `$ARGUMENTS` at the end
3. Add `"./commands/<command-name>.md"` to the `commands` array in `plugin.json`

Output the created file content and confirm the plugin.json update.

If the plugin doesn't exist, suggest running `/new-plugin` first.
