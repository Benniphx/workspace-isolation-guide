# Workspace Isolation - Setup Instructions

> Technical step-by-step guide. For background and diagrams see [README.md](./README.md)

---

## Requirements

| Tool | Version | Installation | Purpose |
|------|---------|--------------|---------|
| jj (Jujutsu) | 0.20+ | `brew install jj` | VCS with workspace support |
| Claude Code | 1.0+ | homebrew or npm | AI coding assistant |
| mise | latest | `brew install mise` | Tool version manager (optional) |
| uv | 0.9+ | via mise | Python package manager (optional) |
| jq | latest | `brew install jq` | JSON processing (optional, for metadata) |
| sqlite3 | built-in | macOS built-in | System dependency |

---

## Step 1: Initialize jj in Your Project

```bash
cd your-project

# If starting fresh:
jj git init --colocate

# If already has .git:
jj git init --git-repo=.
```

---

## Step 2: Install workspace-claude Script

```bash
mkdir -p ~/.local/bin
curl -o ~/.local/bin/workspace-claude \
  https://raw.githubusercontent.com/Benniphx/workspace-isolation-guide/main/workspace-claude
chmod +x ~/.local/bin/workspace-claude
```

The script auto-detects project type (Python/Node) and sets up the workspace accordingly.

**Output on launch:**
```
  setup ·· cleanup ·· create ·· fetch ·· sync ·· uv sync ·· memory ·· ready

  ┌ myproject ─────────────────────────────────
  │ Session    20260211151717-e94c (main)
  │ Workspace  /path/to/workspace
  │ Memory     global symlink (4 files)
  │ Python     3.13.6
  │ jj         @kpqvuntx (on main)
  │ CLAUDE.md  ✓
  │ Tools      mise 2026.2.9, direnv 2.37.1
  └────────────────────────────────────────
```

---

## Step 3: Add Shell Function

Add to `~/.zshrc` (or `~/.bashrc`):

```bash
# Resolve Claude session UUID → workspace path via sessions-index.json
_resolve_claude_session() {
    local uuid="$1"
    command -v jq &>/dev/null || return 1
    for index in ~/.claude/projects/*/sessions-index.json; do
        [[ -f "$index" ]] || continue
        local path
        path=$(jq -r --arg id "$uuid" \
            '[.entries[] | select(.sessionId == $id) | .projectPath] | first // empty' \
            "$index" 2>/dev/null)
        if [[ -n "$path" && "$path" != "null" && -d "$path" ]]; then
            echo "$path"
            return 0
        fi
    done
    return 1
}

# Projects that use workspace isolation (paths relative to $HOME)
WORKSPACE_ISOLATED_PROJECTS=(
    "projects/myproject"
    # Add more projects here
)

# Auto-redirect claude to workspace-claude for isolated projects
claude() {
    # --- UUID-Resume: auto-resolve workspace from Claude session UUID ---
    local resume_uuid=""
    local -a args=("$@")
    local i
    for ((i=1; i<=${#args[@]}; i++)); do
        if [[ "${args[$i]}" == "--resume" || "${args[$i]}" == "-r" ]]; then
            local next=$((i + 1))
            if [[ $next -le ${#args[@]} ]] && [[ "${args[$next]}" =~ ^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$ ]]; then
                resume_uuid="${args[$next]}"
            fi
            break
        fi
    done

    if [[ -n "$resume_uuid" ]]; then
        local ws_path
        ws_path=$(_resolve_claude_session "$resume_uuid")
        if [[ -n "$ws_path" && -d "$ws_path" ]]; then
            echo "🔄 Resuming in workspace: $(basename "$ws_path")"
            cd "$ws_path"
            eval "$(mise activate bash 2>/dev/null)" || true
            direnv allow 2>/dev/null || true
            command claude --dangerously-skip-permissions "$@"
            return
        fi
    fi
    # --- End UUID-Resume ---

    local home_relative="${PWD#$HOME/}"

    for project in "${WORKSPACE_ISOLATED_PROJECTS[@]}"; do
        if [[ "$home_relative" == "$project"* ]] || [[ "$home_relative" == "${project}-workspaces"* ]]; then
            # Already in a workspace? Use claude directly
            if [[ "$home_relative" == *"-workspaces"* ]]; then
                command claude "$@"
                return
            fi
            # In project root - spawn workspace
            echo "🚀 Workspace isolation active - launching isolated session..."
            workspace-claude "$@"
            return
        fi
    done

    # Not an isolated project - use normal claude with auto-accept permissions
    # Remove --dangerously-skip-permissions if you prefer the trust dialog
    command claude --dangerously-skip-permissions "$@"
}
```

> **Note:** The `--dangerously-skip-permissions` flag skips the permission/trust dialog.
> This is safe for your own projects where you trust the CLAUDE.md and settings files.
> Remove this flag if you work on untrusted codebases.

After saving:

```bash
source ~/.zshrc
```

---

## Step 4: Verify PATH

Ensure `~/.local/bin` is in your PATH:

```bash
# Add to ~/.zshrc if not present
export PATH="$HOME/.local/bin:$PATH"
```

---

## Usage

### Normal Session (based on main)

```bash
cd ~/projects/myproject
claude
# → Automatically creates isolated workspace
```

### Session on Specific Branch

```bash
workspace-claude --branch fix/issue-123-bugfix
```

### Session Resume

```bash
workspace-claude --sessions                   # List sessions with UUIDs
workspace-claude --resume a8f5                # Resume by workspace prefix
workspace-claude --resume 18fcd521-0457-...   # Resume by Claude session UUID
claude --resume 18fcd521-0457-...             # Auto-resolves workspace from UUID
```

### Workspace Management

```bash
workspace-claude --list              # Show active workspaces
workspace-claude --cleanup           # Remove old workspaces (workday-based)
workspace-claude --remove session-20260121143052-a3f2
```

---

## Directory Structure After Setup

```
~/projects/
├── myproject/                         ← Main directory
│   ├── .jj/
│   ├── src/
│   └── ...
│
└── myproject-workspaces/              ← Auto-created
    ├── session-20260121143052-a3f2/
    │   ├── .jj/  (→ points to main repo)
    │   ├── src/
    │   └── .venv/  (own!)
    └── session-20260121150823-b7c9/
        └── ...
```

---

## Troubleshooting

### "fatal: not a git repository"

Normal - jj workspaces aren't git-colocated. Ignore this warning.

### Workspace Setup Takes Long

First start creates `.venv` / `node_modules`. Subsequent starts are faster.

### Debug Mode

```bash
bash -x ~/.local/bin/workspace-claude 2>&1 | head -50
```

### Manual Cleanup

```bash
rm -rf ~/Documents/project-workspaces/session-*
# Then clean up jj:
cd ~/Documents/project && jj workspace list
# For each dead workspace:
jj workspace forget session-XXXXX
```

### `claude --resume` Fails: "command not found: head"

**Symptom:** `claude --resume <UUID>` fails with `_resolve_claude_session:8: command not found: head` (repeated 8x).

**Cause:** Powerlevel10k Instant Prompt redirects stdout/stderr during zsh init, while PATH is not yet fully set. `head` is an external command in `/usr/bin/` and unavailable at that point.

**Fix in `~/.zshrc`** (`_resolve_claude_session` function, ~line 164-166):

```zsh
# Before (external command dependency):
path=$(jq -r --arg id "$uuid" \
    '.entries[] | select(.sessionId == $id) | .projectPath' \
    "$index" 2>/dev/null | head -1)

# After (jq-native, no external dependency):
path=$(jq -r --arg id "$uuid" \
    '[.entries[] | select(.sessionId == $id) | .projectPath] | first // empty' \
    "$index" 2>/dev/null)
```

**Why:** jq-native `first // empty` replaces `head -1` entirely -- no external dependency in the pipeline, robust against PATH issues during p10k Instant Prompt.

### Wrong Claude Version

If mise-managed node shadows system Claude:

```bash
rm -rf ~/.local/share/mise/installs/node/*/lib/node_modules/@anthropic-ai/claude-code
rm -f ~/.local/share/mise/installs/node/*/bin/claude
```

---

## Optional: Beads Integration

If your project uses [Beads](https://github.com/...) for issue tracking, issues are managed centrally from the main repo:

```bash
# Automatically in workspace-claude:
export BEADS_DIR="$PROJECT_ROOT/.beads"
```

---

*Last updated: 2026-02-17 (v2.3.0)*
