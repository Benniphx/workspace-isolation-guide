# Workspace Isolation - Setup Instructions

> Technical step-by-step guide. For background and diagrams see [workspace-isolation-guide.md](./workspace-isolation-guide.md)

---

## Requirements

| Tool | Version | Installation | Purpose |
|------|---------|--------------|---------|
| jj (Jujutsu) | 0.20+ | `brew install jj` | VCS with workspace support |
| Claude Code | 1.0+ | homebrew or npm | AI coding assistant |
| mise | latest | `brew install mise` | Tool version manager (optional) |
| uv | 0.9+ | via mise | Python package manager (optional) |

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
# workspace-claude - Launch Claude in isolated jj workspace
#
# Usage: workspace-claude [options] [claude-args...]
#
# Options:
#   --branch <rev>    Base workspace on specific revision (default: main)
#   --list            List active workspaces
#   --cleanup         Remove workspaces older than 24h
#   --remove <name>   Remove specific workspace

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
MAX_AGE_HOURS=24

# IMPORTANT: All logs to stderr, never stdout!
# stdout is reserved for returning the workspace path
log_info() { echo -e "\033[0;32m[workspace]\033[0m $1" >&2; }
log_warn() { echo -e "\033[1;33m[workspace]\033[0m $1" >&2; }
log_error() { echo -e "\033[0;31m[workspace]\033[0m $1" >&2; }

cleanup_old_workspaces() {
    [[ -d "$WORKSPACES_DIR" ]] || return 0

    local count=0
    local now=$(date +%s)
    local max_age_seconds=$((MAX_AGE_HOURS * 3600))

    for ws in "$WORKSPACES_DIR"/session-*; do
        [[ -d "$ws" ]] || continue
        local mtime=$(stat -f %m "$ws" 2>/dev/null || stat -c %Y "$ws" 2>/dev/null)
        local age=$((now - mtime))

        if [[ $age -gt $max_age_seconds ]]; then
            local ws_name=$(basename "$ws")
            log_info "Removing old workspace: $ws_name"
            (cd "$PROJECT_ROOT" && jj workspace forget "$ws_name" 2>/dev/null || true)
            rm -rf "$ws"
            ((count++)) || true
        fi
    done

    [[ $count -gt 0 ]] && log_info "Cleaned up $count workspace(s)"
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

# Parse arguments
BASE_REV="main"
ACTION="run"
REMOVE_TARGET=""
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
        --cleanup)
            ACTION="cleanup"
            shift
            ;;
        --remove)
            ACTION="remove"
            REMOVE_TARGET="$2"
            shift 2
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
# Projects that use workspace isolation (paths relative to $HOME)
WORKSPACE_ISOLATED_PROJECTS=(
    "projects/myproject"
    # Add more projects here
)

# Auto-redirect claude to workspace-claude for isolated projects
claude() {
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

    # Not an isolated project - use normal claude
    command claude "$@"
}
```

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

### Workspace Management

```bash
workspace-claude --list              # Show active workspaces
workspace-claude --cleanup           # Remove old workspaces (>24h)
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

*Last updated: 2026-01-21*
