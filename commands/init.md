---
name: init
description: Install language and security rules to ~/.claude/rules/. Self-destructs after running so it reappears when the plugin updates with new rules.
---

Install language and security rules from this plugin to the user's global Claude Code rules directory.

## Steps

1. **Show available rules** - List all `.md` files in this plugin's `rules/` directory (relative to `${CLAUDE_PLUGIN_ROOT}`), showing which ones already exist in `~/.claude/rules/` and which are new or updated.

2. **Detect conflicts** - For each rule that already exists in `~/.claude/rules/`, diff the plugin version against the installed version. Categorize each file as:
   - **New** - doesn't exist yet
   - **Unchanged** - identical content, no action needed
   - **Conflict** - both exist with different content

3. **Ask which to install** - Use AskUserQuestion to let the user choose:
   - "Install all" (recommended) - copies everything, overwrites conflicts with plugin version
   - "Install new only" - skip files that already exist in ~/.claude/rules/
   - "Review conflicts" - for each conflicting file, show a summary of differences and ask: overwrite, skip, or merge (append plugin-only sections)

4. **Handle conflicts** - When "Review conflicts" is selected:
   - For each conflicting file, read both versions and summarize the key differences
   - Use AskUserQuestion per file: "Overwrite with plugin version", "Keep existing", or "Merge (keep both, deduplicate)"
   - For merge: combine both files, remove duplicate sections, preserve user customizations that don't exist in the plugin version

5. **Copy rules** - For each selected rule file:
   - Create `~/.claude/rules/languages/` if it doesn't exist
   - Copy rule files preserving the directory structure (e.g., `rules/languages/python.md` → `~/.claude/rules/languages/python.md`)
   - Report what was installed, skipped, and merged

6. **Self-destruct** - After successful installation:
   - Find this plugin's cache directory by looking for `rules` plugin in `~/.claude/plugins/cache/`
   - Delete ONLY this `init.md` command file from the cached copy: `~/.claude/plugins/cache/*/rules/commands/init.md`
   - Explain to the user: "The init command has been removed from your local cache. It will reappear next time the rules plugin updates, prompting you to install any new or changed rules."

7. **Summary** - Show what was installed, skipped, and merged. Remind the user to restart Claude Code for rules to take effect.

## Important

- The source rules live at `${CLAUDE_PLUGIN_ROOT}/rules/` - read from there
- The destination is `~/.claude/rules/` - write there
- Preserve subdirectory structure (e.g., `languages/`)
- When comparing existing vs new, use file content diff, not just existence
- The self-destruct targets the CACHE copy, not the source repo
