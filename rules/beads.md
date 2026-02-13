---
paths:
  - "**/.beads/**"
---

# Beads Issue Tracking Standards

## Issue Classification

When creating beads issues, **MUST** immediately label each issue as either `needs-design` or `ready-to-implement`.

### `needs-design` — Requires design before coding

Label an issue `needs-design` when ANY of these apply:

- Multiple valid implementation approaches exist
- Architectural decisions needed (patterns, data flow, component structure)
- Changes affect 3+ files in a coordinated way
- Involves restructuring, decomposing, or merging existing code
- Trade-offs between approaches need user input
- Touches interfaces, protocols, or public APIs
- Requires choosing between competing patterns (e.g., `asyncio.wait` vs wrapper pattern)

### `ready-to-implement` — Mechanical, no ambiguity

Label an issue `ready-to-implement` when ALL of these apply:

- Single clear implementation path
- Scope is contained (1-2 files, localized change)
- No design decisions required
- Examples: extract duplicate code, add caching, fix deprecated API call, move inline import to module level, add memoization

### Workflow

```bash
# Create and immediately label
bd create --title="Fix X" --type=bug --priority=P2
bd label add <id> needs-design  # or ready-to-implement
```

When creating issues in bulk, label all issues in the same pass — do not defer labeling to a separate step.

## Troubleshooting

### Database Corruption (the "no such column" special)

When `bd` commands fail with SQLite schema errors like:

```
Error: failed to open database: failed to initialize schema: sqlite3: SQL logic error: no such column: spec_id
```

This is a known issue caused by CLI version upgrades against an older DB schema. The schema changed but the database didn't get the memo.

**Key fact:** JSONL is the source of truth, not SQLite. The database is a derived index. Deleting it is like clearing your browser history — cathartic and consequence-free.

**Recovery steps (try in order):**

1. **Try** the built-in doctor first: `bd doctor --fix --force --source=jsonl`
   - `--force` bypasses the broken database open
   - `--source=jsonl` rebuilds from the canonical JSONL files
   - **Caveat:** This often fails when the DB can't open at all (the `spec_id` error blocks even the fix command)
2. **If doctor fails** (it usually does for schema errors), the nuclear option works every time:

   ```bash
   rm -f .beads/beads.db .beads/beads.db-wal .beads/beads.db-shm
   bd list  # triggers DB rebuild from JSONL
   bd migrate --update-repo-id  # stamps repo fingerprint on fresh DB
   ```

3. Verify recovery: `bd list` or `bd stats`

**Note:** After rebuilding, `bd` may warn about a "LEGACY DATABASE" until you run `bd migrate --update-repo-id`. This is expected — the fresh DB has no repo fingerprint yet.

**Rules:**

- **MUST NOT** attempt to manually ALTER TABLE the SQLite schema. That way lies madness
- **MUST NOT** treat the SQLite database as precious data — JSONL files are what matter
- **SHOULD** run `bd doctor` after CLI version upgrades to catch schema drift early
- **SHOULD** check `bd version` when schema errors appear — version mismatches are almost always the cause
