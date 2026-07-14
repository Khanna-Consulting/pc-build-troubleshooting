# Tmux Setup and Keyboard Shortcuts

## Config File

Located at `~/.tmux.conf` on the kc-llm server.

```bash
# Switch sessions with Ctrl+b + number
bind 1 switch-client -t NBA
bind 2 switch-client -t test
bind 3 switch-client -t main

# Quick session picker with Ctrl+s (no prefix needed)
bind -n C-s choose-session
```

## Custom Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+s` | Open session picker (no prefix needed) |
| `Ctrl+b` then `1` | Switch to NBA session |
| `Ctrl+b` then `2` | Switch to test session |
| `Ctrl+b` then `3` | Switch to main session |

## Default Tmux Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+b` then `s` | Session picker (default) |
| `Ctrl+b` then `c` | Create new window |
| `Ctrl+b` then `n` | Next window |
| `Ctrl+b` then `p` | Previous window |
| `Ctrl+b` then `d` | Detach from session |
| `Ctrl+b` then `%` | Split pane vertically |
| `Ctrl+b` then `"` | Split pane horizontally |
| `Ctrl+b` then arrow keys | Move between panes |
| `Ctrl+b` then `x` | Kill current pane |
| `Ctrl+b` then `,` | Rename current window |
| `Ctrl+b` then `$` | Rename current session |

## Common Commands

```bash
# Create a new session
tmux new -s <name>

# List sessions
tmux ls

# Attach to a session (from outside tmux)
tmux attach -t <name>

# Kill a session
tmux kill-session -t <name>

# Reload config after changes
tmux source-file ~/.tmux.conf
```
