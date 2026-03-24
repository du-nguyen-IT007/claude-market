---
description: Scan all git submodule plugins, pull latest, and sync versions into marketplace.json
---

You are managing the `claude-market` plugin marketplace. Your task is to scan all git submodule plugins, update them to their latest commits, and keep `marketplace.json` in sync.

## Steps

### 1. Discover all submodules

Run:
```bash
git submodule status
```

List every submodule path and its current commit SHA.

### 2. Pull latest for each submodule

Run:
```bash
git submodule update --remote --merge
```

Then re-run `git submodule status` to capture updated SHAs.

### 3. Read each plugin's manifest

For each submodule path (e.g. `plugins/superpowers`), read `.claude-plugin/plugin.json` to extract:
- `name`
- `version`
- `description`
- `author`
- `homepage`
- `repository`
- `license`
- `keywords`

If `.claude-plugin/plugin.json` does not exist, warn the user and skip that submodule.

### 4. Sync marketplace.json

Read `.claude-plugin/marketplace.json`.

For each plugin entry whose `source` points to a submodule path:
- Update `version` to match the submodule's `plugin.json`
- Update any other fields that differ (`description`, `keywords`, etc.)

For submodule plugins not yet in `marketplace.json`, append a new entry with `source` set to the submodule path (e.g. `"./plugins/superpowers"`).

Write the updated `.claude-plugin/marketplace.json`.

### 5. Stage and commit

Run:
```bash
git add .gitmodules plugins/ .claude-plugin/marketplace.json
```

Show a summary diff, then ask the user: **"Commit these changes? (y/n)"**

If yes, commit with message:
```
chore: sync submodule plugins to latest

<list each plugin: name vOLD → vNEW or "new">
```

### 6. Report

Print a table:

| Plugin | Old version | New version | Status |
|--------|-------------|-------------|--------|
| superpowers | 5.0.4 | 5.0.5 | updated |
| plugin-creator | — | — | local (skipped) |

Local plugins (non-submodule) are noted as skipped — only submodules are updated.
