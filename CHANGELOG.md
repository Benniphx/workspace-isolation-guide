# workspace-claude Changelog

## Compatibility

**Tested with Claude Code:** 2.1.19
**Last checked:** 2026-02-11

### Platform Support

| Platform | Status | Notes |
|----------|--------|-------|
| macOS (ARM/Intel) | ✅ Tested | Primary development platform |
| Linux (Arch/CachyOS) | ✅ Compatible | All commands have GNU fallbacks |
| Linux (Ubuntu/Debian) | ✅ Compatible | Tested with GNU coreutils |
| WSL2 | 🔄 Should work | Not explicitly tested |

### Linux Requirements (Arch/CachyOS)

```bash
# Install dependencies
sudo pacman -S jj git jq sqlite

# Optional: mise for tool management
curl https://mise.run | sh

# Install Claude Code
# (Check Anthropic docs for Linux installation)
```

### Cross-Platform Commands

The script handles macOS vs Linux differences:

| Command | macOS | Linux (GNU) |
|---------|-------|-------------|
| stat mtime | `stat -f %m` | `stat -c %Y` |
| date from timestamp | `date -r $ts` | `date -d @$ts` |
| date parse | `date -j -f "%Y-%m-%d"` | `date -d "$date"` |
| sed in-place | `sed -i.bak` | `sed -i.bak` (compatible) |

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

## [2.3.0] - 2026-02-11

### Added
- Enhanced startup summary via `print_startup_summary()` function
- Shows project name, session ID, base revision, and optional UUID
- Shows memory status (symlinked/local/not configured + file count)
- Shows Python/Node environment status with version
- Shows jj change-id and bookmark info
- Shows CLAUDE.md presence
- Shows mise/direnv tool versions

### Changed
- `run` and `resume_session()` both use shared `print_startup_summary()` instead of separate header blocks
- Unified output format: all lines use consistent `emoji + aligned label + value` style
- Removed multi-line header blocks from both run and resume paths
- Fixed Tools line comma spacing (was missing space after comma)

---

## [2.2.0] - 2026-02-11

### Added
- UUID-based session resume: `workspace-claude --resume <claude-session-uuid>`
- Shell function `_resolve_claude_session()` in `~/.zshrc` - resolves UUID to workspace path
- `resolve_session_uuid()` in workspace-claude script
- `--sessions` now shows Claude session UUID (truncated to 8 chars)
- `claude --resume <uuid>` auto-resolves workspace and cd's there before resuming

### Changed
- `resume_session()` now accepts both workspace prefix and Claude session UUID
- When resuming by UUID, passes `--resume <uuid>` to Claude for exact session restore
- Help text updated with UUID examples

---

## [2.1.0] - 2026-02-10

### Added
- Auto-recovery from stale jj working copy in `create_workspace`, `cleanup_old_workspaces`
- `--ignore-working-copy` fallback for `jj git fetch` and `jj bookmark set`
- `--version` flag showing workspace-claude and Claude Code versions

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
