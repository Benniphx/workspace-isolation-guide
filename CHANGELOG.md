# workspace-claude Changelog

## Compatibility

**Tested with Claude Code:** 2.1.19
**Last checked:** 2026-01-26

### Claude Code Features We Rely On

| Feature | Since | Our Usage |
|---------|-------|-----------|
| `--resume` flag | v1.x | Used in `resume_session()` |
| `--dangerously-skip-permissions` | v1.x | Used for trusted workspaces |
| `sessions-index.json` format | v2.x | Parsed in `list_sessions()` |
| Project-based session storage | v2.x | Scanned for cross-workspace discovery |

### Claude Code Fixes That Affect Us

| Version | Fix | Impact |
|---------|-----|--------|
| 2.1.19 | Fixed `/rename` and `/tag` in different directories | No action needed - native fix |
| 2.1.19 | Fixed resuming by title from different directory | No action needed - native fix |

---

## [2.0.0] - 2026-01-26

### Added
- `--sessions` - List all workspace sessions with filtering (excludes memory-observer, sub-agents)
- `--resume <prefix>` - Resume session by workspace prefix
- `--keep <prefix>` - Protect session from automatic cleanup
- `--unkeep <prefix>` - Remove protection
- `--help` - Full help text
- `KEEP_WORKDAYS` environment variable for workday-based cleanup
- `.session-meta.json` metadata file per workspace
- `is_user_session()` filter function to exclude:
  - Memory-observer sessions ("Claude-Mem", "memory agent")
  - Sub-agent sessions ("observed_from_primary", "task-notification")
  - System messages ("PROGRESS SUMMARY")
  - Short sessions (< 3 messages)
- Cross-workspace session discovery via `~/.claude/projects/` scanning

### Changed
- Cleanup now uses workday logic (Mon-Fri) instead of fixed 24h
- Friday sessions survive Monday cleanup
- `list_sessions()` shows only most recent session per workspace

### Technical Details

**Session Discovery:**
- Scans `~/.claude/projects/-*-workspaces-session-*/sessions-index.json`
- Parses JSON with jq (fallback without jq available)
- Filters by `firstPrompt` content and `messageCount`

**Metadata Format (.session-meta.json):**
```json
{
  "created": "2026-01-26T08:30:02Z",
  "last_active": "2026-01-26T09:15:00Z",
  "description": "",
  "keep": false,
  "session_id": "session-20260126083002-a8f5"
}
```

---

## [1.0.0] - 2026-01-21

### Initial Release
- `--branch <rev>` - Base workspace on specific revision
- `--list` - List active workspaces with age
- `--cleanup` - Remove old workspaces (24h)
- `--remove <name>` - Remove specific workspace
- Auto-detection of project root (.jj or .git)
- Python (uv sync) and Node.js (npm install) project setup
- Environment file copying (.envrc, .env, .env.local)
- Beads integration for central issue management

### Bug Fixes (2026-01-21)
- Fixed stdout pollution from log functions (redirect to stderr)
- Fixed unbound variable crash with empty CLAUDE_ARGS array
- Fixed session ID collision (8 chars → 19 chars)

---

## Migration Notes

### From 1.x to 2.0

1. **No breaking changes** - existing workspaces continue to work
2. **New metadata** - created automatically on new workspaces
3. **Existing workspaces** - can add metadata via `--keep` command

### Future Claude Code Updates

If Claude Code adds native workspace/session management, check:

1. **Session storage location** - Do they still use `~/.claude/projects/<path>/`?
2. **sessions-index.json format** - Any schema changes?
3. **--resume behavior** - Does it work across directories now?
4. **Environment variables** - Any new ones for session management?

Run `claude --version` and check CHANGELOG.md at:
https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
