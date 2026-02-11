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
| sqlite3 | built-in | macOS built-in | For claude-mem session queries |

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

cat > ~/.local/bin/workspace-claude << 'EOF'
#!/bin/bash
# workspace-claude - Launch Claude in isolated jj workspace (v2.0)
#
# Usage: workspace-claude [options] [claude-args...]
#
# Options:
#   --branch <rev>    Base workspace on specific revision (default: main)
#   --list            List active workspaces with age
#   --sessions        List sessions with claude-mem context
#   --resume <id>     Resume by prefix or Claude session UUID
#   --keep <prefix>   Protect session from cleanup
#   --unkeep <prefix> Remove protection
#   --cleanup         Remove old workspaces (respects workday logic)
#   --remove <name>   Remove specific workspace
#   --help            Show help

set -euo pipefail

# Auto-detect project root (first parent with .jj or .git)
find_project_root() {
    local dir="$PWD"
    while [[ "$dir" != "/" ]]; do
        if [[ -d "$dir/.jj" ]] || [[ -d "$dir/.git" ]]; then
            echo "$dir"
            return 0
        fi
        dir="$(dirname "$dir")"
    done
    echo "$PWD"
}

PROJECT_ROOT="$(find_project_root)"
PROJECT_NAME="$(basename "$PROJECT_ROOT")"
WORKSPACES_DIR="${PROJECT_ROOT}-workspaces"
KEEP_WORKDAYS="${KEEP_WORKDAYS:-2}"  # Keep sessions from last N workdays (Mon-Fri)

# IMPORTANT: All logs to stderr, never stdout!
# stdout is reserved for returning the workspace path
log_info() { echo -e "\033[0;32m[workspace]\033[0m $1" >&2; }
log_warn() { echo -e "\033[1;33m[workspace]\033[0m $1" >&2; }
log_error() { echo -e "\033[0;31m[workspace]\033[0m $1" >&2; }

# Calculate cutoff date based on workdays (Mon-Fri)
calculate_cutoff_date() {
    local keep_days="${1:-$KEEP_WORKDAYS}"
    local now_ts=$(date +%s)
    local days_to_subtract=0
    local workdays_counted=0

    while [[ $workdays_counted -lt $keep_days ]]; do
        ((days_to_subtract++))
        local check_ts=$((now_ts - days_to_subtract * 86400))
        local check_dow=$(date -r "$check_ts" +%u 2>/dev/null || date -d "@$check_ts" +%u 2>/dev/null)
        if [[ $check_dow -ge 1 && $check_dow -le 5 ]]; then
            ((workdays_counted++))
        fi
    done

    local cutoff_ts=$((now_ts - days_to_subtract * 86400))
    local cutoff_date=$(date -r "$cutoff_ts" +%Y-%m-%d 2>/dev/null || date -d "@$cutoff_ts" +%Y-%m-%d 2>/dev/null)
    date -j -f "%Y-%m-%d" "$cutoff_date" +%s 2>/dev/null || date -d "$cutoff_date" +%s 2>/dev/null
}

# Read session metadata field
read_session_meta() {
    local workspace_path="$1"
    local field="$2"
    local meta_file="${workspace_path}/.session-meta.json"

    if [[ ! -f "$meta_file" ]]; then
        echo ""
        return
    fi

    if command -v jq &>/dev/null; then
        jq -r ".$field // empty" "$meta_file" 2>/dev/null
    else
        grep -o "\"$field\": *\"[^\"]*\"" "$meta_file" 2>/dev/null | sed 's/.*: *"\([^"]*\)"/\1/' || \
        grep -o "\"$field\": *[^,}]*" "$meta_file" 2>/dev/null | sed 's/.*: *//' | tr -d ' '
    fi
}

cleanup_old_workspaces() {
    [[ -d "$WORKSPACES_DIR" ]] || return 0

    local count=0
    local skipped=0
    local cutoff_ts
    cutoff_ts=$(calculate_cutoff_date "$KEEP_WORKDAYS")

    log_info "Cleanup: keeping sessions from last $KEEP_WORKDAYS workday(s)"

    for ws in "$WORKSPACES_DIR"/session-*; do
        [[ -d "$ws" ]] || continue
        local ws_name=$(basename "$ws")

        # Check keep flag
        local keep_flag=$(read_session_meta "$ws" "keep")
        if [[ "$keep_flag" == "true" ]]; then
            log_info "Keeping (protected): $ws_name"
            ((skipped++)) || true
            continue
        fi

        local last_active=$(read_session_meta "$ws" "last_active")
        local ws_ts

        if [[ -n "$last_active" ]]; then
            ws_ts=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$last_active" +%s 2>/dev/null || \
                    date -d "$last_active" +%s 2>/dev/null || \
                    stat -f %m "$ws" 2>/dev/null || stat -c %Y "$ws" 2>/dev/null)
        else
            ws_ts=$(stat -f %m "$ws" 2>/dev/null || stat -c %Y "$ws" 2>/dev/null)
        fi

        if [[ $ws_ts -lt $cutoff_ts ]]; then
            log_info "Removing old workspace: $ws_name"
            (cd "$PROJECT_ROOT" && jj workspace forget "$ws_name" 2>/dev/null || true)
            rm -rf "$ws"
            ((count++)) || true
        fi
    done

    [[ $count -gt 0 ]] && log_info "Cleaned up $count workspace(s)"
    [[ $skipped -gt 0 ]] && log_info "Skipped $skipped protected workspace(s)"
    return 0
}

create_workspace() {
    local session_id="$1"
    local base_rev="${2:-main}"
    # IMPORTANT: 19 chars = YYYYMMDDHHMMSS-xxxx (prevents same-day collisions)
    local short_id="${session_id:0:19}"
    local workspace_name="session-${short_id}"
    local workspace_path="${WORKSPACES_DIR}/${workspace_name}"

    mkdir -p "$WORKSPACES_DIR"
    cleanup_old_workspaces

    if [[ -d "$workspace_path" ]]; then
        log_info "Workspace exists: $workspace_name"
        echo "$workspace_path"
        return 0
    fi

    log_info "Creating workspace: $workspace_name (based on $base_rev)"

    cd "$PROJECT_ROOT"
    log_info "Fetching latest..."
    jj git fetch 2>/dev/null || log_warn "Could not fetch"

    jj workspace add "$workspace_path" -r "$base_rev" 2>/dev/null || {
        log_error "Failed to create workspace"
        return 1
    }

    # Copy environment files
    for f in .envrc .env .env.local; do
        [[ -f "$PROJECT_ROOT/$f" ]] && cp "$PROJECT_ROOT/$f" "$workspace_path/"
    done

    # Create session metadata
    cat > "${workspace_path}/.session-meta.json" << EOFMETA
{
  "created": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "last_active": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "description": "",
  "keep": false,
  "session_id": "$workspace_name"
}
EOFMETA

    cd "$workspace_path"

    # Setup based on project type
    # IMPORTANT: Output to /dev/null to prevent stdout pollution!
    if [[ -f "pyproject.toml" ]]; then
        log_info "Python project - running uv sync..."
        mise trust "$workspace_path" 2>/dev/null || true
        eval "$(mise activate bash 2>/dev/null)" || true
        uv sync >/dev/null 2>&1 || log_warn "uv sync failed"
    elif [[ -f "package.json" ]]; then
        log_info "Node project - running npm install..."
        npm install >/dev/null 2>&1 || log_warn "npm install failed"
    fi

    log_info "Workspace ready: $workspace_path"
    echo "$workspace_path"
}

list_workspaces() {
    echo "Active workspaces for $PROJECT_NAME:"
    echo "======================================"

    if [[ ! -d "$WORKSPACES_DIR" ]]; then
        echo "  (none)"
        return
    fi

    for ws in "$WORKSPACES_DIR"/session-*; do
        [[ -d "$ws" ]] || continue
        local ws_name=$(basename "$ws")
        local mtime=$(stat -f %m "$ws" 2>/dev/null || stat -c %Y "$ws" 2>/dev/null)
        local age_hours=$(( ($(date +%s) - mtime) / 3600 ))
        echo "  $ws_name (age: ${age_hours}h)"
    done

    echo ""
    echo "jj workspaces:"
    (cd "$PROJECT_ROOT" && jj workspace list 2>/dev/null) || echo "  (not a jj repo)"
}

remove_workspace() {
    local workspace_name="$1"
    local workspace_path="${WORKSPACES_DIR}/${workspace_name}"

    if [[ ! -d "$workspace_path" ]]; then
        log_error "Workspace not found: $workspace_name"
        return 1
    fi

    log_info "Removing: $workspace_name"
    (cd "$PROJECT_ROOT" && jj workspace forget "$workspace_name" 2>/dev/null || true)
    rm -rf "$workspace_path"
    log_info "Done"
}

# Find workspace by prefix
find_workspace_by_prefix() {
    local prefix="$1"
    local matches=()

    [[ -d "$WORKSPACES_DIR" ]] || return 1

    for ws in "$WORKSPACES_DIR"/session-*"$prefix"*; do
        [[ -d "$ws" ]] && matches+=("$ws")
    done

    if [[ ${#matches[@]} -eq 0 ]]; then
        log_error "No workspace found matching: $prefix"
        return 1
    elif [[ ${#matches[@]} -gt 1 ]]; then
        log_error "Multiple workspaces match '$prefix':"
        for m in "${matches[@]}"; do
            echo "  $(basename "$m")" >&2
        done
        return 1
    fi

    echo "${matches[0]}"
}

# Keep/unkeep session
keep_session() {
    local prefix="$1"
    local keep_value="${2:-true}"

    local workspace_path
    workspace_path=$(find_workspace_by_prefix "$prefix") || return 1

    local meta_file="${workspace_path}/.session-meta.json"
    if command -v jq &>/dev/null && [[ -f "$meta_file" ]]; then
        local tmp_file=$(mktemp)
        jq ".keep = $keep_value | .last_active = \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"" "$meta_file" > "$tmp_file"
        mv "$tmp_file" "$meta_file"
    fi

    if [[ "$keep_value" == "true" ]]; then
        log_info "Protected: $(basename "$workspace_path")"
    else
        log_info "Unprotected: $(basename "$workspace_path")"
    fi
}

# Resume session
resume_session() {
    local prefix="$1"
    shift
    local claude_args=("$@")

    local workspace_path
    workspace_path=$(find_workspace_by_prefix "$prefix") || return 1

    local session_id=$(basename "$workspace_path")

    echo "🔄 Resuming Workspace Session"
    echo "   Project: $PROJECT_NAME"
    echo "   Session: ${session_id#session-}"
    echo ""
    echo "📁 Workspace: $workspace_path"

    if [[ -d "$PROJECT_ROOT/.beads" ]]; then
        export BEADS_DIR="$PROJECT_ROOT/.beads"
        echo "📋 Beads: $BEADS_DIR (central)"
    fi

    echo ""

    cd "$workspace_path"
    eval "$(mise activate bash 2>/dev/null)" || true
    direnv allow 2>/dev/null || true

    exec command claude --dangerously-skip-permissions --resume ${claude_args[@]+"${claude_args[@]}"}
}

# Parse arguments
BASE_REV="main"
ACTION="run"
REMOVE_TARGET=""
RESUME_PREFIX=""
KEEP_PREFIX=""
CLAUDE_ARGS=()

while [[ $# -gt 0 ]]; do
    case "$1" in
        --branch)
            BASE_REV="$2"
            shift 2
            ;;
        --list)
            ACTION="list"
            shift
            ;;
        --sessions)
            ACTION="sessions"
            shift
            ;;
        --resume)
            ACTION="resume"
            RESUME_PREFIX="$2"
            shift 2
            ;;
        --keep)
            ACTION="keep"
            KEEP_PREFIX="$2"
            shift 2
            ;;
        --unkeep)
            ACTION="unkeep"
            KEEP_PREFIX="$2"
            shift 2
            ;;
        --cleanup)
            ACTION="cleanup"
            shift
            ;;
        --remove)
            ACTION="remove"
            REMOVE_TARGET="$2"
            shift 2
            ;;
        --help|-h)
            echo "Usage: workspace-claude [options] [claude-args...]"
            echo ""
            echo "Options:"
            echo "  --branch <rev>    Base on revision (default: main)"
            echo "  --list            List workspaces"
            echo "  --sessions        List sessions with context"
            echo "  --resume <prefix> Resume session"
            echo "  --keep <prefix>   Protect session"
            echo "  --unkeep <prefix> Unprotect session"
            echo "  --cleanup         Remove old workspaces"
            echo "  --remove <name>   Remove workspace"
            echo "  --help            Show this help"
            exit 0
            ;;
        *)
            CLAUDE_ARGS+=("$1")
            shift
            ;;
    esac
done

case "$ACTION" in
    list)
        list_workspaces
        ;;
    sessions)
        echo "Session listing requires claude-mem integration."
        echo "Use --list for basic workspace info."
        list_workspaces
        ;;
    resume)
        resume_session "$RESUME_PREFIX" "${CLAUDE_ARGS[@]}"
        ;;
    keep)
        keep_session "$KEEP_PREFIX" "true"
        ;;
    unkeep)
        keep_session "$KEEP_PREFIX" "false"
        ;;
    cleanup)
        cleanup_old_workspaces
        ;;
    remove)
        remove_workspace "$REMOVE_TARGET"
        ;;
    run)
        session_id="$(date +%Y%m%d%H%M%S)-$(openssl rand -hex 4)"

        echo "🚀 Workspace Claude Launcher"
        echo "   Project: $PROJECT_NAME"
        echo "   Session: ${session_id:0:19}"
        echo "   Base: $BASE_REV"
        echo ""

        workspace_path=$(create_workspace "$session_id" "$BASE_REV")

        if [[ ! -d "$workspace_path" ]]; then
            echo "❌ Failed to create workspace"
            exit 1
        fi

        echo ""
        echo "📁 Workspace: $workspace_path"

        # Set BEADS_DIR if project has beads
        if [[ -d "$PROJECT_ROOT/.beads" ]]; then
            export BEADS_DIR="$PROJECT_ROOT/.beads"
            echo "📋 Beads: $BEADS_DIR (central)"
        fi

        echo ""

        cd "$workspace_path"
        eval "$(mise activate bash 2>/dev/null)" || true
        direnv allow 2>/dev/null || true

        # Skip trust dialog - workspaces are created from trusted repo
        # IMPORTANT: ${arr[@]+"${arr[@]}"} prevents "unbound variable" with empty arrays
        exec command claude --dangerously-skip-permissions ${CLAUDE_ARGS[@]+"${CLAUDE_ARGS[@]}"}
        ;;
esac
EOF

chmod +x ~/.local/bin/workspace-claude
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
            '.entries[] | select(.sessionId == $id) | .projectPath' \
            "$index" 2>/dev/null | head -1)
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

*Last updated: 2026-02-11*
