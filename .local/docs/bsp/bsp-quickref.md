# BSP - Quick Reference

**Version**: 2.0.0
**Pattern**: Git-like CLI Dispatcher + Unified Daemon

---

## Quick Start

```bash
# Start daemon
bsp daemon start

# Enable features
bsp balance on
bsp desktops on
bsp ventilate on

# Check status
bsp status

# Get help
bsp help
bsp help <command>
```

---

## Command Syntax

```
bsp <command> [subcommand] [options] [args]
```

**Global Options**:
- `--help`, `-h`: Show help
- `--version`, `-v`: Show version
- `--debug`: Enable debug mode
- `--verbose`: Verbose output

---

## Daemon Management

```bash
bsp daemon start              # Start daemon
bsp daemon stop               # Stop daemon
bsp daemon restart            # Restart daemon
bsp daemon status             # Check status
bsp daemon reload             # Reload configuration
```

---

## Status & Information

```bash
bsp status                    # Show overall status
bsp status --json             # JSON output
bsp status --verbose          # Verbose status

bsp tree                      # Show window tree (all desktops)
bsp tree --focused            # Focused desktop only
bsp tree --desktop 2          # Specific desktop
bsp tree --json               # JSON output
```

---

## Feature Management

```bash
# List features
bsp feature list              # All features
bsp feature list --enabled    # Enabled only
bsp feature list --disabled   # Disabled only

# Control features
bsp feature enable <name>     # Enable feature
bsp feature disable <name>    # Disable feature
bsp feature status <name>     # Check status
```

---

## Balance Feature

```bash
bsp balance on                # Enable auto-balance
bsp balance off               # Disable auto-balance
bsp balance status            # Check status
bsp balance now               # Balance immediately
bsp balance now focused       # Balance focused desktop
bsp balance config            # Show configuration
```

**Configuration**:
```json
{
  "features": {
    "balance": {
      "enabled": true,
      "strategy": "equal",      // equal | golden
      "exclude_classes": [],
      "debounce_ms": 100
    }
  }
}
```

---

## Desktops Feature

```bash
bsp desktops on               # Enable dynamic desktops
bsp desktops off              # Disable dynamic desktops
bsp desktops status           # Check status
bsp desktops config           # Show configuration
bsp desktops config start_id 5    # Set start ID
bsp desktops config min_empty 2   # Set min empty desktops
```

**Configuration**:
```json
{
  "features": {
    "desktops": {
      "enabled": true,
      "start_id": 1,
      "min_empty": 1,
      "auto_remove": true
    }
  }
}
```

---

## Flag Feature

```bash
bsp flag on                   # Enable flag handling
bsp flag off                  # Disable flag handling
bsp flag status               # Check status
bsp flag config               # Show configuration
bsp flag config marked 3 blue # Set marked border
```

**Supported Flags**:
- `marked`: Window marked state
- `floating`: Window floating state
- `hidden`: Window hidden state
- `locked`: Window locked state
- `sticky`: Window sticky state
- `private`: Window private state

**Configuration**:
```json
{
  "features": {
    "flag": {
      "enabled": true,
      "handlers": {
        "marked": {
          "border_width": 3,
          "border_color": "#0000ff"
        },
        "floating": { "enabled": true },
        "hidden": { "enabled": true },
        "locked": { "enabled": true },
        "sticky": { "enabled": true },
        "private": { "enabled": true }
      }
    }
  }
}
```

---

## Floating Borders Feature

```bash
bsp floating-borders on       # Enable
bsp floating-borders off      # Disable
bsp floating-borders status   # Check status
bsp floating-borders config 0 # Set border width
```

**Configuration**:
```json
{
  "features": {
    "floating-borders": {
      "enabled": true,
      "border_width": 0
    }
  }
}
```

---

## Ventilate Feature (Terminal Swallowing)

```bash
bsp ventilate on              # Enable ventilate
bsp ventilate off             # Disable ventilate
bsp ventilate status          # Check status
bsp ventilate list            # List configuration

# Manage terminal classes
bsp ventilate add-terminal kitty
bsp ventilate remove-terminal kitty

# Manage protected classes (won't be swallowed)
bsp ventilate add-no-inhale zoom
bsp ventilate remove-no-inhale zoom
```

**Configuration**:
```json
{
  "features": {
    "ventilate": {
      "enabled": true,
      "terminals": [
        "Alacritty",
        "kitty",
        "terminal-drop",
        "Code"
      ],
      "no_inhale": [
        "Rofi",
        "xev",
        "firefox",
        "zoom"
      ]
    }
  }
}
```

---

## Direct Window Operations

```bash
# Directional window placement
bsp side west                 # Place window west
bsp side east                 # Place window east
bsp side north                # Place window north
bsp side south                # Place window south

# Hide/show windows
bsp shroud cover              # Hide all windows + polybar
bsp shroud reveal             # Show all windows + polybar
bsp shroud toggle             # Toggle cover/reveal

# Fullscreen
bsp fullscreen                # Toggle fullscreen
bsp fullscreen on             # Enter fullscreen
bsp fullscreen off            # Exit fullscreen
```

---

## Layout Management

```bash
# List layouts
bsp layout list               # List all saved layouts

# Save current layout
bsp layout save dev           # Save as "dev"
bsp layout save work --description "Work setup"

# Load layout
bsp layout load dev           # Load to current desktop
bsp layout apply focused dev  # Apply to focused desktop
bsp layout apply 2 media      # Apply to desktop 2

# Manage layouts
bsp layout delete old         # Delete layout
bsp layout export dev dev.json    # Export to file
bsp layout import dev.json mydev  # Import from file

# Interactive selector
bsp layout select             # Rofi layout selector
```

**Layout File** (`~/.config/bsp/layouts/dev.json`):
```json
{
  "name": "dev",
  "description": "Development layout",
  "tree": {
    "type": "split",
    "splitType": "vertical",
    "splitRatio": 0.618,
    "firstChild": {
      "type": "window",
      "class": "Code"
    },
    "secondChild": {
      "type": "split",
      "splitType": "horizontal",
      "splitRatio": 0.5,
      "firstChild": { "type": "window", "class": "Alacritty" },
      "secondChild": { "type": "window", "class": "Alacritty" }
    }
  }
}
```

---

## Desktop Operations

```bash
bsp desktop list              # List desktops
bsp desktop add work          # Add desktop
bsp desktop remove 5          # Remove desktop
bsp desktop rename 2 browser  # Rename desktop
bsp desktop focus 3           # Focus desktop
bsp desktop swap 1 2          # Swap desktops
bsp desktop select            # Rofi desktop selector
```

---

## Monitor Management

```bash
bsp monitor list              # List monitors
bsp monitor focus eDP1        # Focus monitor
bsp monitor rename eDP1 laptop    # Rename monitor

# Monitor profiles
bsp monitor profile list      # List profiles
bsp monitor profile save work # Save current setup
bsp monitor profile load work # Load profile
bsp monitor profile detect    # Auto-detect and apply
```

---

## Node Operations

```bash
bsp node focus <selector>     # Focus node
bsp node close                # Close focused node
bsp node close <selector>     # Close specific node
bsp node kill                 # Kill focused node
bsp node swap <sel1> <sel2>   # Swap nodes
bsp node move <sel> <desktop> # Move to desktop
bsp node rotate 90            # Rotate tree 90°
bsp node flip vertical        # Flip tree vertically
bsp node balance              # Balance subtree
```

---

## Query Operations

```bash
bsp query nodes               # Query all nodes
bsp query nodes .window       # Query all windows
bsp query nodes .focused      # Query focused node
bsp query desktops            # Query all desktops
bsp query desktops .occupied  # Query occupied desktops
bsp query monitors            # Query all monitors
bsp query tree                # Query tree structure
```

---

## Window Rules

```bash
bsp rule list                 # List all rules
bsp rule add "class=firefox desktop=^2"
bsp rule remove <id>          # Remove rule
bsp rule test <window-id>     # Test rule matching
```

---

## Desktop Snapshots

```bash
bsp snapshot save             # Save current desktop
bsp snapshot save backup      # Save with name
bsp snapshot restore backup   # Restore snapshot
bsp snapshot list             # List snapshots
bsp snapshot delete old       # Delete snapshot
```

---

## Configuration Management

```bash
# Get/set configuration
bsp config get <key>
bsp config get features.balance.enabled
bsp config set <key> <value>
bsp config set features.balance.enabled true

# Manage configuration
bsp config list               # List all config
bsp config edit               # Open in editor
bsp config validate           # Validate config
bsp config reset              # Reset to defaults
bsp config reset <key>        # Reset specific key
```

**Configuration File**: `~/.config/bsp/config.json`

---

## Help & Version

```bash
bsp help                      # General help
bsp help <command>            # Command-specific help
bsp help balance              # Balance feature help
bsp --version                 # Show version
bsp version --verbose         # Detailed version info
```

---

## Common Workflows

### Initial Setup

```bash
# 1. Start daemon
bsp daemon start

# 2. Enable desired features
bsp balance on
bsp desktops on
bsp ventilate on

# 3. Configure features
bsp desktops config start_id 1
bsp ventilate add-terminal kitty

# 4. Check status
bsp status
```

### Save and Apply Layout

```bash
# 1. Arrange windows as desired
# 2. Save layout
bsp layout save my-workspace

# 3. Later, load layout
bsp layout load my-workspace

# Or use interactive selector
bsp layout select
```

### Debug Issues

```bash
# 1. Check daemon status
bsp daemon status

# 2. View tree structure
bsp tree --focused

# 3. Check feature status
bsp status --verbose

# 4. View logs
tail -f ~/.local/state/bsp/daemon.log

# 5. Restart daemon
bsp daemon restart
```

### Configure Ventilate

```bash
# 1. Add your terminal emulator
bsp ventilate add-terminal kitty

# 2. Add apps that shouldn't be swallowed
bsp ventilate add-no-inhale zoom
bsp ventilate add-no-inhale slack

# 3. Verify configuration
bsp ventilate list
```

---

## Event Types (for Advanced Usage)

When developing custom features or debugging:

- `node_add`: New window created
- `node_remove`: Window destroyed
- `node_state`: Window state changed (tiled, floating, fullscreen)
- `node_flag`: Window flag changed (marked, locked, sticky, hidden, private)
- `node_geometry`: Window size/position changed
- `node_focus`: Focus changed
- `desktop_add`: Desktop created
- `desktop_remove`: Desktop removed
- `desktop_focus`: Desktop focus changed
- `monitor_add`: Monitor connected
- `monitor_remove`: Monitor disconnected
- `monitor_focus`: Monitor focus changed

---

## File Locations

**Configuration**:
- Main config: `~/.config/bsp/config.json`
- Layouts: `~/.config/bsp/layouts/`
- Rules: `~/.config/bsp/rules.json`
- Profiles: `~/.config/bsp/profiles/`

**State**:
- Daemon PID: `~/.local/state/bsp/daemon.pid`
- Unix socket: `$XDG_RUNTIME_DIR/bsp.sock`
- Ventilate state: `~/.local/state/bsp/ventilate-state.json`
- Snapshots: `~/.local/state/bsp/snapshots/`

**Cache**:
- Query cache: `~/.cache/bsp/query-cache.json`
- Event log: `~/.cache/bsp/event-log.jsonl`

**Logs**:
- Daemon log: `~/.local/state/bsp/daemon.log`

**Binaries**:
- Main CLI: `~/.local/bin/bsp`
- Daemon: `~/.local/libexec/bsp/bspd`
- Helpers: `~/.local/libexec/bsp/bsp-*`
- Features: `~/.local/libexec/bsp/features/*.zsh`

**Systemd**:
- Service: `~/.local/share/systemd/user/bsp.service`

---

## Systemd Integration

```bash
# Enable on login
systemctl --user enable bsp.service

# Start manually
systemctl --user start bsp.service

# Check status
systemctl --user status bsp.service

# View logs
journalctl --user -u bsp.service -f

# Stop
systemctl --user stop bsp.service

# Disable
systemctl --user disable bsp.service
```

---

## Tab Completion

### Zsh

Add to `~/.zshrc`:
```zsh
# bsp completion
if [[ -f ~/.local/share/zsh/completions/_bsp ]]; then
    fpath=(~/.local/share/zsh/completions $fpath)
fi
```

### Bash

Add to `~/.bashrc`:
```bash
# bsp completion
if [[ -f ~/.local/share/bash-completion/completions/bsp ]]; then
    source ~/.local/share/bash-completion/completions/bsp
fi
```

---

## Keybinding Examples (sxhkd)

```bash
# ~/.config/sxhkd/sxhkdrc

# Directional placement
super + shift + {h,j,k,l}
    bsp side {west,south,north,east}

# Fullscreen
super + f
    bsp fullscreen

# Shroud
super + s
    bsp shroud toggle

# Layout selector
super + shift + l
    bsp layout select

# Tree view
super + shift + t
    alacritty -e bsp tree

# Balance now
super + shift + b
    bsp balance now
```

---

## Troubleshooting

### Daemon Won't Start

```bash
# Check if already running
bsp daemon status

# Check logs
tail -f ~/.local/state/bsp/daemon.log

# Check bspwm is running
pgrep bspwm

# Try foreground mode
bsp daemon start --foreground --debug
```

### Feature Not Working

```bash
# Check feature status
bsp feature status <name>

# Check feature configuration
bsp config get features.<name>

# Enable feature
bsp feature enable <name>

# Restart daemon
bsp daemon restart
```

### Tree Not Displaying

```bash
# Check bspwm connection
bspc query -T -m

# Try JSON output
bsp tree --json

# Check permissions
ls -l ~/.local/libexec/bsp/bsp-tree
```

### Configuration Issues

```bash
# Validate configuration
bsp config validate

# Check for syntax errors
jq . ~/.config/bsp/config.json

# Reset to defaults
bsp config reset

# View current configuration
bsp config list
```

---

## Performance Tips

1. **Enable Caching**: Reduces bspc calls
   ```json
   {
     "cache": {
       "enabled": true,
       "ttl": { "nodes": 60, "desktops": 60 }
     }
   }
   ```

2. **Debounce Events**: Reduce event processing
   ```json
   {
     "features": {
       "balance": {
         "debounce_ms": 200
       }
     }
   }
   ```

3. **Exclude Classes**: Don't balance certain windows
   ```json
   {
     "features": {
       "balance": {
         "exclude_classes": ["vlc", "mpv", "gimp"]
       }
     }
   }
   ```

4. **Selective Features**: Only enable what you need
   ```bash
   bsp feature disable ventilate  # If not using terminal swallowing
   ```

---

## Migration from Old bsp

### Automatic Migration

```bash
# Migrate configuration
bsp config migrate

# Verify migration
bsp config validate
bsp status
```

### Manual Migration

**Old ventilate config**:
- `~/.config/bsp-ventilate/terminals` ’ `config.json features.ventilate.terminals[]`
- `~/.config/bsp-ventilate/no_inhale` ’ `config.json features.ventilate.no_inhale[]`

**Old systemd services**:
```bash
# Disable old services
systemctl --user disable bsp-balance.service
systemctl --user disable bsp-desktops.service
systemctl --user disable bsp-flag.service
systemctl --user disable bsp-floating-borders.service
systemctl --user disable bsp-ventilate.service

# Enable new service
systemctl --user enable bsp.service
systemctl --user start bsp.service
```

**Old keybindings** (update `~/.config/sxhkd/sxhkdrc`):
```bash
# Old: bsp-side west
# New: bsp side west

# Old: bsp-shroud cover
# New: bsp shroud cover

# Old: bsp-fullscreen
# New: bsp fullscreen
```

---

## Resources

- **Documentation**: `~/.local/docs/bsp/`
- **Man Pages**: `man bsp`, `man bspd`, `man bsp-balance`
- **Examples**: `~/.local/docs/bsp/examples/`
- **Issue Tracker**: https://github.com/username/dotfiles/issues

---

**Quick Reference v1.0**
**Date**: 2025-11-07
**For bsp version**: 2.0.0+
