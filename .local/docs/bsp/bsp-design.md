# BSP - Complete Rewrite Design Specification

## Executive Summary

### Overview
The bsp package rewrite transforms 8 standalone bspwm utilities into a unified, modern window management system with a git-like CLI dispatcher and single daemon architecture.

### Architecture Pattern
**Unified Daemon + CLI Dispatcher** (Inspired by systemd and git patterns)

### Key Design Decisions
1. **Single Daemon**: Replace 5 independent daemons with one unified daemon (bspd)
2. **Event Bus Architecture**: Centralized event routing with feature-based handlers
3. **Git-like CLI**: Unified `bsp <subcommand>` interface with consistent patterns
4. **Plugin-style Features**: Modular feature system (can enable/disable at runtime)
5. **IPC-based Communication**: CLI communicates with daemon via Unix socket
6. **Library-first Design**: Heavy use of library abstractions for reusability

### Library Integration Strategy
- **Core Infrastructure**: _log, _singleton, _lifecycle, _events, _config, _cache
- **bspwm Abstraction**: _bspwm (NEW - high priority library to create)
- **UI Layer**: _rofi, _ui, _notify
- **Layout Management**: _layout (NEW - medium priority library to create)

### Implementation Timeline
- **Phase 1** (Week 1): Foundation - Core infrastructure, _bspwm library, basic daemon
- **Phase 2** (Week 2): Migration - All features migrated to new architecture
- **Phase 3** (Week 3): Enhancement - IPC, status, tree, improved features
- **Phase 4** (Week 4): Advanced - Layout management, Rofi integration, polish

### Resource Requirements
- **Development Time**: 4 weeks full-time
- **Dependencies**: bspwm, bspc, xprop, Zsh, systemd
- **New Libraries Needed**: _bspwm (required), _layout (optional)

---

## Architecture Overview

### Design Pattern: Unified Daemon + CLI Dispatcher

This architecture combines the best of:
- **systemd pattern**: Single daemon with multiple subsystems
- **git pattern**: Unified CLI with subcommand routing
- **Event-driven**: Reactive feature handlers subscribed to event bus
- **Plugin architecture**: Features loaded dynamically, configurable at runtime

### Rationale

**Why Unified Daemon?**
- **Single Event Loop**: One `bspc subscribe` process instead of 5
- **Shared State**: All features access common cache and configuration
- **Coordinated Actions**: Features can interact (e.g., layout + balance)
- **Resource Efficiency**: ~80% reduction in process overhead
- **Simplified Management**: One systemd service instead of 5

**Why Git-like CLI?**
- **Discoverability**: `bsp help` shows all capabilities
- **Consistency**: Uniform command patterns across features
- **Extensibility**: Easy to add new subcommands
- **Familiar**: Developers understand git-like interfaces

**Why Event Bus?**
- **Decoupling**: Features don't depend on each other
- **Testability**: Can inject mock events for testing
- **Flexibility**: Easy to add/remove event handlers
- **Performance**: Can debounce/batch events centrally

### Complete File Structure

```
/home/andronics/.pkgs/bsp/
├── .local/
│   ├── bin/
│   │   └── bsp                              # Main CLI dispatcher
│   ├── libexec/
│   │   └── bsp/
│   │       ├── bspd                         # Main daemon (event loop + IPC)
│   │       ├── bsp-common                   # Shared utilities
│   │       │
│   │       ├── bsp-daemon                   # Daemon control (start/stop/restart/status)
│   │       ├── bsp-status                   # Status reporting (features, windows, desktops)
│   │       ├── bsp-feature                  # Feature control (enable/disable/config)
│   │       │
│   │       ├── bsp-side                     # Side placement logic (CLI utility)
│   │       ├── bsp-shroud                   # Shroud cover/reveal (CLI utility)
│   │       ├── bsp-fullscreen               # Fullscreen toggle (CLI utility)
│   │       │
│   │       ├── bsp-layout                   # Layout management (list/save/load/apply)
│   │       ├── bsp-workspace                # Workspace management (future feature)
│   │       ├── bsp-tree                     # Tree visualization
│   │       ├── bsp-config                   # Config management (get/set/edit)
│   │       │
│   │       ├── bsp-help                     # Help system
│   │       ├── bsp-version                  # Version info
│   │       │
│   │       └── features/                    # Feature modules
│   │           ├── balance.zsh              # Auto-balance feature
│   │           ├── desktops.zsh             # Dynamic desktop management
│   │           ├── flag.zsh                 # Flag change reactions
│   │           ├── floating-borders.zsh     # Remove borders from floating windows
│   │           └── ventilate.zsh            # Terminal swallowing
│   │
│   ├── share/
│   │   └── systemd/
│   │       └── user/
│   │           └── bsp.service              # Single systemd service
│   │
│   └── docs/
│       └── bsp/
│           ├── bsp-design.md                # This document
│           ├── bsp-design-summary.md        # Executive summary
│           ├── bsp-quickref.md              # Quick reference
│           ├── bsp-analysis.md              # Analysis document
│           └── man/
│               ├── bsp.1                    # Main man page
│               ├── bsp-balance.1            # Feature-specific pages
│               └── ...
│
├── .config/
│   └── bsp/
│       ├── config.json                      # Main configuration
│       └── layouts/                         # Layout definitions
│           ├── default.json
│           ├── dev.json
│           ├── media.json
│           └── work.json
│
└── .cache/
    └── bsp/
        ├── cache.json                       # Query result cache
        └── state.json                       # Runtime state
```

### Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
├─────────────────────────────────────────────────────────────┤
│  - CLI interface (bsp + libexec commands)                   │
│  - Rofi menus (interactive selection)                       │
│  - Terminal UI (tree visualization, status display)         │
│  - Notifications (desktop events, errors)                   │
│  - Output formatting (human-readable, JSON, colored)        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
├─────────────────────────────────────────────────────────────┤
│  - Command handlers (side, shroud, fullscreen, layout)      │
│  - Feature orchestration (balance, desktops, flag, etc.)    │
│  - Configuration management (load, validate, update)        │
│  - State management (caching, session state)                │
│  - IPC server/client (Unix socket communication)            │
│  - Event routing (dispatch events to features)              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                       DOMAIN LAYER                           │
├─────────────────────────────────────────────────────────────┤
│  - bspwm operations (_bspwm library)                        │
│  - Layout operations (_layout library)                      │
│  - Event operations (_events library)                       │
│  - Lifecycle operations (_lifecycle library)                │
│  - Window tree logic (queries, transformations)             │
│  - Desktop management logic (add, remove, naming)           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  INFRASTRUCTURE LAYER                        │
├─────────────────────────────────────────────────────────────┤
│  - bspc command execution (query, subscribe, node, desktop) │
│  - xprop command execution (window properties)              │
│  - File system operations (config, cache, state files)      │
│  - Unix socket operations (IPC transport)                   │
│  - Process management (daemon lifecycle, singleton)         │
│  - Logging infrastructure (_log library)                    │
└─────────────────────────────────────────────────────────────┘
```

### Design Principles

1. **Modularity**: Features are independent, self-contained modules
2. **Separation of Concerns**: Clear boundaries between layers
3. **Single Responsibility**: Each component has one well-defined purpose
4. **Configuration over Code**: User preferences in config files, not scripts
5. **Fail Gracefully**: Errors don't crash daemon, features degrade independently
6. **Observable**: Rich status, logging, and debugging capabilities
7. **Testable**: Mock external dependencies, test features in isolation
8. **Extensible**: Easy to add new features, commands, and integrations
9. **Library-first**: Reusable components in libraries, not inline code
10. **User-centric**: Prioritize discoverability, help, and clear feedback

---

## Command Structure

### Command Hierarchy

```
bsp
├── Global Flags
│   ├── -h, --help              # Show help
│   ├── -v, --version           # Show version
│   ├── --debug                 # Enable debug output
│   └── --config <file>         # Use alternate config file
│
├── Daemon Management
│   └── daemon <subcommand>
│       ├── start               # Start daemon
│       ├── stop                # Stop daemon
│       ├── restart             # Restart daemon
│       ├── status              # Show daemon status
│       └── reload              # Reload configuration
│
├── Feature Control (Interactive/Daemon Features)
│   ├── balance <control>
│   │   ├── on                  # Enable balance
│   │   ├── off                 # Disable balance
│   │   ├── status              # Show status
│   │   ├── trigger             # Manually trigger balance
│   │   └── config <key> [val]  # Get/set config
│   │
│   ├── desktops <control>
│   │   ├── on                  # Enable auto-desktops
│   │   ├── off                 # Disable auto-desktops
│   │   ├── status              # Show status
│   │   └── config <key> [val]  # Get/set config (start_id)
│   │
│   ├── flag <control>
│   │   ├── on                  # Enable flag reactions
│   │   ├── off                 # Disable flag reactions
│   │   ├── status              # Show status
│   │   └── config <key> [val]  # Get/set config (colors, borders)
│   │
│   ├── floating-borders <control>
│   │   ├── on                  # Enable floating border removal
│   │   ├── off                 # Disable floating border removal
│   │   └── status              # Show status
│   │
│   └── ventilate <control>
│       ├── on                  # Enable terminal swallowing
│       ├── off                 # Disable terminal swallowing
│       ├── status              # Show status
│       ├── config add-terminal <class>      # Add terminal class
│       ├── config add-no-inhale <class>     # Add no-inhale class
│       ├── config remove-terminal <class>   # Remove terminal class
│       ├── config remove-no-inhale <class>  # Remove no-inhale class
│       └── config list                      # List current config
│
├── Direct Commands (No daemon needed)
│   ├── side <direction>
│   │   ├── west                # Place window west
│   │   ├── north               # Place window north
│   │   ├── east                # Place window east
│   │   └── south               # Place window south
│   │
│   ├── shroud <action>
│   │   ├── cover               # Hide polybar and windows
│   │   └── reveal              # Show polybar and windows
│   │
│   └── fullscreen              # Toggle fullscreen
│
├── Layout Management
│   └── layout <subcommand>
│       ├── list                # List available layouts
│       ├── save <name>         # Save current layout
│       ├── load <name>         # Load layout
│       ├── apply <name>        # Apply layout to focused desktop
│       ├── apply <desktop> <name>  # Apply to specific desktop
│       ├── delete <name>       # Delete layout
│       ├── export <name> <file>    # Export layout to file
│       ├── import <file>       # Import layout from file
│       └── select              # Interactive Rofi selector
│
├── Workspace Management (Future)
│   └── workspace <subcommand>
│       ├── create <name>       # Create named workspace
│       ├── switch <name>       # Switch to workspace
│       ├── list                # List workspaces
│       ├── delete <name>       # Delete workspace
│       ├── rename <old> <new>  # Rename workspace
│       └── select              # Interactive Rofi selector
│
├── Utilities
│   ├── tree [options]          # Show window tree
│   │   ├── --focused           # Show focused desktop only
│   │   ├── --desktop <id>      # Show specific desktop
│   │   ├── --color             # Colored output
│   │   └── --json              # JSON output
│   │
│   ├── status [options]        # Show comprehensive status
│   │   ├── --json              # JSON output
│   │   └── --watch             # Continuous updates
│   │
│   └── config <subcommand>
│       ├── get <key>           # Get config value
│       ├── set <key> <value>   # Set config value
│       ├── edit                # Open config in editor
│       ├── validate            # Validate config
│       ├── path                # Show config file path
│       └── reload              # Reload config (daemon)
│
└── Help
    ├── help [command]          # Show help for command
    └── --version               # Show version info
```

### Command Specifications

#### Global Commands

##### `bsp --help`
**Purpose**: Display general help and list all available commands

**Signature**:
```bash
bsp --help
bsp -h
bsp help [command]
```

**Returns**:
- 0: Success
- 1: Unknown command (when specific command requested)

**Output**: Help text with command listing and descriptions

**Example**:
```bash
$ bsp --help
BSP - Binary Space Partitioning Window Manager Utilities

Usage: bsp <command> [options] [args]

Daemon Management:
  daemon start|stop|restart|status  Control the bsp daemon
  status                            Show comprehensive status

Feature Control:
  balance on|off|status             Auto-balance window tree
  desktops on|off|status            Dynamic desktop management
  flag on|off|status                React to window flag changes
  floating-borders on|off|status    Remove borders from floating windows
  ventilate on|off|status           Terminal window swallowing

Direct Commands:
  side <direction>                  Place window directionally
  shroud cover|reveal               Hide/show windows and polybar
  fullscreen                        Toggle fullscreen

Layout Management:
  layout list|save|load|apply       Manage layout profiles

Utilities:
  tree [options]                    Visualize window tree
  config get|set|edit               Manage configuration

Global Options:
  -h, --help                        Show this help
  -v, --version                     Show version
  --debug                           Enable debug output
  --config <file>                   Use alternate config

See 'bsp help <command>' for more information on a specific command.
```

**Implementation**: Main dispatcher inline, calls bsp-help for specific commands

**Library Integration**: _args for parsing

---

##### `bsp --version`
**Purpose**: Display version information

**Signature**:
```bash
bsp --version
bsp -v
```

**Returns**: 0 (always)

**Output**: Version string with build info

**Example**:
```bash
$ bsp --version
bsp version 2.0.0
Build: 2025-11-07
bspwm version: 0.9.10
```

**Implementation**: Main dispatcher inline, calls bsp-version

**Library Integration**: None (simple version string)

---

#### Daemon Management

##### `bsp daemon <subcommand>`
**Purpose**: Control daemon lifecycle

**Signature**:
```bash
bsp daemon start      # Start daemon
bsp daemon stop       # Stop daemon
bsp daemon restart    # Restart daemon
bsp daemon status     # Check daemon status
bsp daemon reload     # Reload configuration
```

**Parameters**:
- `subcommand`: One of start, stop, restart, status, reload (required)

**Returns**:
- 0: Success
- 1: Daemon not running (for stop/restart/reload/status)
- 2: Daemon already running (for start)
- 3: Failed to start/stop

**Output**: Status message

**Examples**:
```bash
$ bsp daemon start
Starting bsp daemon... [OK]
Daemon running (PID 12345)

$ bsp daemon status
Daemon: RUNNING (PID 12345)
Uptime: 2 hours 34 minutes
Socket: /run/user/1000/bsp.sock

$ bsp daemon stop
Stopping bsp daemon... [OK]
```

**Implementation**: libexec/bsp/bsp-daemon script
- Manages systemd service OR direct daemon process
- Uses _singleton for PID tracking
- IPC to daemon for status/reload

**Library Integration**:
- _singleton: Check/set PID file
- _lifecycle: Start/stop daemon
- _log: Logging

---

##### `bsp status [--json] [--watch]`
**Purpose**: Show comprehensive system status

**Signature**:
```bash
bsp status [--json] [--watch]
```

**Parameters**:
- `--json`: Output as JSON (optional)
- `--watch`: Continuous updates (optional)

**Returns**:
- 0: Success
- 1: Daemon not running

**Output**: Status display (human-readable or JSON)

**Example** (human-readable):
```bash
$ bsp status

BSP Window Manager Utilities
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Daemon: RUNNING (PID 12345)
Uptime: 2 hours 34 minutes
Socket: /run/user/1000/bsp.sock

Features:
  ✓ balance           ENABLED
  ✓ desktops          ENABLED  (start_id: 1)
  ✓ flag              ENABLED
  ✓ floating-borders  ENABLED
  ✓ ventilate         ENABLED  (3 terminals, 6 no-inhale)

Window Manager:
  Windows: 12
  Desktops: 5 (2 occupied, 3 empty)
  Monitors: 2 (eDP-1, HDMI-1)
  Focused: Desktop 1, Window 0x00400001

Layouts:
  - default
  - dev
  - media
  - work

Configuration:
  File: /home/andronics/.config/bsp/config.json
  Last modified: 2 hours ago
  Valid: ✓
```

**Example** (JSON):
```bash
$ bsp status --json
{
  "daemon": {
    "status": "running",
    "pid": 12345,
    "uptime": 9240,
    "socket": "/run/user/1000/bsp.sock"
  },
  "features": {
    "balance": {"enabled": true},
    "desktops": {"enabled": true, "config": {"start_id": 1}},
    "flag": {"enabled": true},
    "floating_borders": {"enabled": true},
    "ventilate": {
      "enabled": true,
      "config": {
        "terminals_count": 3,
        "no_inhale_count": 6
      }
    }
  },
  "window_manager": {
    "windows": 12,
    "desktops": 5,
    "desktops_occupied": 2,
    "monitors": 2,
    "focused_desktop": 1,
    "focused_window": "0x00400001"
  },
  "layouts": ["default", "dev", "media", "work"],
  "config": {
    "file": "/home/andronics/.config/bsp/config.json",
    "last_modified": 7200,
    "valid": true
  }
}
```

**Implementation**: libexec/bsp/bsp-status script
- IPC to daemon for feature status
- Query bspwm for window manager state
- Read config file for layout list
- Format output (human or JSON)

**Library Integration**:
- _bspwm: Query windows, desktops, monitors
- _ui: Format tables and status display
- _config: Read config file

---

#### Feature Control

##### `bsp balance <control>`
**Purpose**: Control auto-balance feature

**Signature**:
```bash
bsp balance on                 # Enable balance
bsp balance off                # Disable balance
bsp balance status             # Show status
bsp balance trigger            # Manually trigger balance
bsp balance config [key] [val] # Get/set config
```

**Parameters**:
- `control`: One of on, off, status, trigger, config (required)
- `key`: Config key (for config subcommand)
- `value`: Config value (for config subcommand)

**Returns**:
- 0: Success
- 1: Daemon not running
- 2: Invalid arguments
- 3: Feature operation failed

**Output**: Status message or config value

**Examples**:
```bash
$ bsp balance on
Feature 'balance' enabled

$ bsp balance status
Feature: balance
Status: ENABLED
Events: node_add, node_remove, node_state, node_geometry
Handled: 1,234 events since start
Last event: 2 seconds ago

$ bsp balance trigger
Balancing all windows... [OK]
Balanced 12 windows on 2 desktops

$ bsp balance config
{
  "enabled": true,
  "strategy": "equal",
  "exclude_classes": []
}
```

**Implementation**: libexec/bsp/bsp-feature script (generic feature control)
- IPC to daemon for enable/disable/status
- Calls feature-specific handler in daemon

**Library Integration**:
- IPC client (Unix socket)
- _config: Read/write config

---

##### `bsp desktops <control>`
**Purpose**: Control dynamic desktop management

**Signature**:
```bash
bsp desktops on                      # Enable auto-desktops
bsp desktops off                     # Disable auto-desktops
bsp desktops status                  # Show status
bsp desktops config <key> [value]    # Get/set config
```

**Parameters**:
- `control`: One of on, off, status, config (required)
- `key`: Config key (start_id) for config subcommand
- `value`: Config value for config subcommand

**Returns**:
- 0: Success
- 1: Daemon not running
- 2: Invalid arguments

**Output**: Status message or config value

**Examples**:
```bash
$ bsp desktops on
Feature 'desktops' enabled

$ bsp desktops status
Feature: desktops
Status: ENABLED
Events: node_add, node_remove, node_state, node_geometry
Desktops: 5 (auto-managed)
Start ID: 1
Empty desktops: 1 (maintained)
Last action: Added desktop 6 (5 seconds ago)

$ bsp desktops config start_id 5
Desktop start ID set to: 5

$ bsp desktops config start_id
5
```

**Implementation**: libexec/bsp/bsp-feature script
- IPC to daemon
- Feature-specific config in daemon

**Library Integration**:
- IPC client
- _config: Config management

---

##### `bsp flag <control>`
**Purpose**: Control window flag reaction feature

**Signature**:
```bash
bsp flag on                      # Enable flag reactions
bsp flag off                     # Disable flag reactions
bsp flag status                  # Show status
bsp flag config <key> [value]    # Get/set config
```

**Parameters**:
- `control`: One of on, off, status, config (required)
- `key`: Config key (marked_border_width, marked_border_color, etc.)
- `value`: Config value

**Returns**:
- 0: Success
- 1: Daemon not running
- 2: Invalid arguments

**Output**: Status message or config value

**Examples**:
```bash
$ bsp flag on
Feature 'flag' enabled

$ bsp flag status
Feature: flag
Status: ENABLED
Events: node_flag
Handled flags:
  ✓ marked (implemented)
  ✓ floating (implemented)
  ✓ hidden (implemented)
  ✓ sticky (implemented)
  ✓ locked (implemented)
  ✓ private (implemented)

$ bsp flag config marked_border_width 3
Marked border width set to: 3

$ bsp flag config
{
  "enabled": true,
  "marked": {
    "border_width": 3,
    "border_color": "#0000ff"
  },
  "floating": {
    "border_width": 2,
    "border_color": "#00ff00"
  }
}
```

**Implementation**: libexec/bsp/bsp-feature script
- IPC to daemon
- Complex config for each flag type

**Library Integration**:
- IPC client
- _config: Config management

---

##### `bsp floating-borders <control>`
**Purpose**: Control floating window border removal

**Signature**:
```bash
bsp floating-borders on     # Enable floating border removal
bsp floating-borders off    # Disable floating border removal
bsp floating-borders status # Show status
```

**Parameters**:
- `control`: One of on, off, status (required)

**Returns**:
- 0: Success
- 1: Daemon not running

**Output**: Status message

**Examples**:
```bash
$ bsp floating-borders on
Feature 'floating-borders' enabled

$ bsp floating-borders status
Feature: floating-borders
Status: ENABLED
Events: node_state
Border width for floating: 0
Handled: 45 state changes since start
```

**Implementation**: libexec/bsp/bsp-feature script
- IPC to daemon
- Simple on/off feature

**Library Integration**:
- IPC client

---

##### `bsp ventilate <control>`
**Purpose**: Control terminal window swallowing

**Signature**:
```bash
bsp ventilate on                                  # Enable swallowing
bsp ventilate off                                 # Disable swallowing
bsp ventilate status                              # Show status
bsp ventilate config add-terminal <class>         # Add terminal class
bsp ventilate config add-no-inhale <class>        # Add no-inhale class
bsp ventilate config remove-terminal <class>      # Remove terminal class
bsp ventilate config remove-no-inhale <class>     # Remove no-inhale class
bsp ventilate config list                         # List current config
```

**Parameters**:
- `control`: One of on, off, status, config (required)
- `config-action`: For config subcommand (add-terminal, add-no-inhale, etc.)
- `class`: Window class name

**Returns**:
- 0: Success
- 1: Daemon not running
- 2: Invalid arguments

**Output**: Status message or config listing

**Examples**:
```bash
$ bsp ventilate on
Feature 'ventilate' enabled

$ bsp ventilate status
Feature: ventilate
Status: ENABLED
Events: node_add, node_remove
Terminals: 3 (Alacritty, terminal-drop, Code)
No-inhale: 6 (Rofi, xev, xprop, gcr-prompter, firefox, Google-chrome)
Currently inhaled: 2 windows
Last action: Inhaled Alacritty (3 seconds ago)

$ bsp ventilate config add-terminal kitty
Added terminal class: kitty

$ bsp ventilate config list
Terminals:
  - Alacritty
  - terminal-drop
  - Code
  - kitty

No-inhale classes:
  - Rofi
  - xev
  - xprop
  - gcr-prompter
  - firefox
  - Google-chrome
```

**Implementation**: libexec/bsp/bsp-feature script
- IPC to daemon for enable/disable/status
- Config stored in main config.json (not separate files)
- Runtime config updates via IPC

**Library Integration**:
- IPC client
- _config: Config management

---

#### Direct Commands

##### `bsp side <direction>`
**Purpose**: Directional window placement with smart tiling

**Signature**:
```bash
bsp side <direction>
```

**Parameters**:
- `direction`: One of west, north, east, south (required)

**Returns**:
- 0: Success
- 1: Invalid direction
- 2: bspwm not running

**Output**: None (silent on success), error message on failure

**Examples**:
```bash
$ bsp side west
# Places focused window on west side with vertical split

$ bsp side north
# Places focused window on north side with horizontal split
```

**Implementation**: libexec/bsp/bsp-side script
- Direct bspwm manipulation (no daemon needed)
- Logic: Focus appropriate split, set orientation, enforce tiled state
- Mirrors current bsp-side behavior exactly

**Library Integration**:
- _bspwm: Query focused node, manipulate nodes, set state
- _log: Logging

---

##### `bsp shroud <action>`
**Purpose**: Hide/show windows and polybar simultaneously

**Signature**:
```bash
bsp shroud cover    # Hide polybar and windows
bsp shroud reveal   # Show polybar and windows
```

**Parameters**:
- `action`: One of cover, reveal (required)

**Returns**:
- 0: Success
- 1: Invalid action
- 2: polybar not running

**Output**: None (silent on success), error message on failure

**Examples**:
```bash
$ bsp shroud cover
# Hides polybar (polybar-msg cmd hide)
# Hides all leaf nodes on focused desktop (bspc node -g hidden=on)

$ bsp shroud reveal
# Shows polybar (polybar-msg cmd show)
# Shows all leaf nodes on focused desktop (bspc node -g hidden=off)
```

**Implementation**: libexec/bsp/bsp-shroud script
- Direct commands (no daemon needed)
- Calls polybar-msg and bspc
- Mirrors current bsp-shroud behavior exactly

**Library Integration**:
- _bspwm: Query nodes, set hidden flag
- _log: Logging

---

##### `bsp fullscreen`
**Purpose**: Toggle fullscreen state for focused window

**Signature**:
```bash
bsp fullscreen
```

**Parameters**: None

**Returns**:
- 0: Success
- 1: No focused window
- 2: bspwm not running

**Output**: None (silent on success), error message on failure

**Example**:
```bash
$ bsp fullscreen
# Toggles between tiled and fullscreen state
```

**Implementation**: libexec/bsp/bsp-fullscreen script
- Direct bspwm command (no daemon needed)
- Simple state toggle
- Mirrors current bsp-fullscreen behavior exactly

**Library Integration**:
- _bspwm: Query state, set state
- _log: Logging

---

#### Layout Management

##### `bsp layout <subcommand>`
**Purpose**: Manage layout profiles

**Signature**:
```bash
bsp layout list                        # List available layouts
bsp layout save <name>                 # Save current layout
bsp layout load <name>                 # Load layout (preview)
bsp layout apply <name>                # Apply to focused desktop
bsp layout apply <desktop> <name>      # Apply to specific desktop
bsp layout delete <name>               # Delete layout
bsp layout export <name> <file>        # Export layout to file
bsp layout import <file>               # Import layout from file
bsp layout select                      # Interactive Rofi selector
```

**Parameters**:
- `subcommand`: One of list, save, load, apply, delete, export, import, select
- `name`: Layout name (required for most subcommands)
- `desktop`: Desktop selector (optional for apply)
- `file`: File path (required for export/import)

**Returns**:
- 0: Success
- 1: Layout not found
- 2: Invalid arguments
- 3: Operation failed

**Output**: Status message or layout listing

**Examples**:
```bash
$ bsp layout list
Available layouts:
  default     - Default balanced layout
  dev         - Development layout (editor + terminals)
  media       - Media layout (player + controls)
  work        - Work layout (browser + apps)

$ bsp layout save my-layout
Layout saved: my-layout
Captured:
  - 5 windows
  - 2 desktops
  - Split ratios: 0.5, 0.618, 0.5

$ bsp layout apply dev
Applying layout 'dev' to focused desktop...
Layout applied successfully
- Created 3 preselections
- Launched 0 applications (manual launch needed)

$ bsp layout select
# Opens Rofi menu with layout list
# User selects layout
# Applies to focused desktop

$ bsp layout export dev ~/layouts/dev-layout.json
Layout exported: ~/layouts/dev-layout.json
```

**Implementation**: libexec/bsp/bsp-layout script
- Direct commands (no daemon needed)
- Uses _layout library for layout operations
- Layout files stored in ~/.config/bsp/layouts/

**Library Integration**:
- _layout: Layout save/load/apply operations
- _bspwm: Query tree structure, create preselections
- _rofi: Interactive selection (for select subcommand)
- _config: Read layout directory path

---

#### Utilities

##### `bsp tree [options]`
**Purpose**: Visualize window tree

**Signature**:
```bash
bsp tree [options]
```

**Parameters**:
- `--focused`: Show focused desktop only (optional)
- `--desktop <id>`: Show specific desktop (optional)
- `--color`: Colored output (optional, default: auto)
- `--json`: JSON output (optional)

**Returns**:
- 0: Success
- 1: Invalid desktop
- 2: bspwm not running

**Output**: ASCII art tree or JSON

**Example** (ASCII):
```bash
$ bsp tree --focused

Desktop 1 (focused)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├─ [V] 0x00400001 (50%)
│  ├─ [T] Code - ~/project/main.rs
│  └─ [H] 0x00400002 (50%)
│     ├─ [T] Alacritty - zsh
│     └─ [T] Alacritty - vim
└─ [F] firefox - Mozilla Firefox

Legend:
  [V] Vertical split   [H] Horizontal split
  [T] Tiled            [F] Floating
  [*] Fullscreen       [M] Marked
```

**Example** (JSON):
```bash
$ bsp tree --focused --json
{
  "desktop": {
    "id": "0x00600001",
    "name": "1",
    "focused": true
  },
  "root": {
    "id": "0x00800001",
    "type": "internal",
    "split_type": "vertical",
    "split_ratio": 0.5,
    "first_child": {
      "id": "0x00400001",
      "type": "window",
      "class": "Code",
      "title": "~/project/main.rs",
      "state": "tiled"
    },
    "second_child": {
      "id": "0x00800002",
      "type": "internal",
      "split_type": "horizontal",
      "split_ratio": 0.5,
      "first_child": {...},
      "second_child": {...}
    }
  }
}
```

**Implementation**: libexec/bsp/bsp-tree script
- Direct query (no daemon needed)
- Queries bspwm tree structure
- Formats as ASCII art or JSON

**Library Integration**:
- _bspwm: Query tree structure
- _ui: Format tree display (ASCII art, colors)

---

##### `bsp config <subcommand>`
**Purpose**: Manage configuration

**Signature**:
```bash
bsp config get <key>            # Get config value
bsp config set <key> <value>    # Set config value
bsp config edit                 # Open config in editor
bsp config validate             # Validate config
bsp config path                 # Show config file path
bsp config reload               # Reload config (daemon)
```

**Parameters**:
- `subcommand`: One of get, set, edit, validate, path, reload
- `key`: Config key (dot notation, e.g., features.balance.enabled)
- `value`: Config value (for set subcommand)

**Returns**:
- 0: Success
- 1: Invalid key
- 2: Invalid value
- 3: Config validation failed

**Output**: Config value, status message, or file path

**Examples**:
```bash
$ bsp config get features.balance.enabled
true

$ bsp config set features.balance.enabled false
Config updated: features.balance.enabled = false
Reloading daemon configuration...

$ bsp config validate
Validating configuration...
✓ Syntax valid (JSON)
✓ Schema valid
✓ All required keys present
✓ No deprecated keys
Configuration is valid

$ bsp config path
/home/andronics/.config/bsp/config.json

$ bsp config edit
Opening /home/andronics/.config/bsp/config.json in $EDITOR...
# (Opens editor)
Configuration saved. Reloading daemon...

$ bsp config reload
Reloading daemon configuration...
Configuration reloaded successfully
```

**Implementation**: libexec/bsp/bsp-config script
- Direct config file operations (for get/edit/validate/path)
- IPC to daemon (for reload, set with immediate reload)

**Library Integration**:
- _config: Config read/write/validate operations
- IPC client (for reload)

---

## Library Integration

### Core Dependencies (Required)

#### _log (Already exists - in use)
**Purpose**: Centralized logging infrastructure

**Current Use**: All utilities use for logging

**Enhanced Use in Rewrite**:
- Structured logging with contexts (feature name, event type)
- Log levels (DEBUG, INFO, WARN, ERROR)
- Daemon logs to centralized location
- Rotation and cleanup

**Functions Used**:
```zsh
log-debug "message"                    # Debug-level logging
log-info "message"                     # Info-level logging
log-warn "message"                     # Warning-level logging
log-error "message"                    # Error-level logging
log-set-level <level>                  # Set log level (DEBUG, INFO, WARN, ERROR)
log-set-context <context>              # Set context (feature name, etc.)
```

**Integration Pattern**:
```zsh
# In daemon
source $(which _log)
log-set-context "bspd"
log-info "Starting daemon"

# In features
log-set-context "bspd:balance"
log-debug "Handling node_add event"
```

---

#### _singleton (Already exists - in use)
**Purpose**: Prevent multiple daemon instances

**Current Use**: All daemons use to enforce single instance

**Enhanced Use in Rewrite**:
- Single daemon singleton (not per-feature)
- PID file tracking
- Graceful handoff on restart

**Functions Used**:
```zsh
_singleton_terminate               # Check and terminate existing instance
_singleton_pid                     # Get PID of existing instance
_singleton_running                 # Check if instance is running
```

**Integration Pattern**:
```zsh
# In daemon (bspd)
source $(which _singleton)
_singleton_terminate && [[ $? -eq 1 ]] && exit 1
# Daemon continues if no existing instance
```

---

#### _lifecycle (May need to create or enhance)
**Purpose**: Feature lifecycle management (start, stop, restart, health)

**Why Needed**: Enable/disable features at runtime, health monitoring

**Functions Needed**:
```zsh
lifecycle-register <feature> <start-fn> <stop-fn>  # Register feature
lifecycle-start <feature>                          # Start feature
lifecycle-stop <feature>                           # Stop feature
lifecycle-restart <feature>                        # Restart feature
lifecycle-status <feature>                         # Check if running
lifecycle-health <feature>                         # Health check
lifecycle-list                                     # List all features
lifecycle-enabled <feature>                        # Check if enabled in config
```

**Integration Pattern**:
```zsh
# In daemon main
source $(which _lifecycle)

# Register features
lifecycle-register "balance" balance_start balance_stop
lifecycle-register "desktops" desktops_start desktops_stop
lifecycle-register "flag" flag_start flag_stop
lifecycle-register "floating-borders" floating_borders_start floating_borders_stop
lifecycle-register "ventilate" ventilate_start ventilate_stop

# Start enabled features
for feature in $(lifecycle-list); do
    if lifecycle-enabled "$feature"; then
        log-info "Starting feature: $feature"
        lifecycle-start "$feature"
    fi
done
```

---

#### _events (May need to create or enhance)
**Purpose**: Centralized event bus for bspwm events

**Why Needed**: Single event loop, route to multiple handlers, debouncing

**Functions Needed**:
```zsh
events-subscribe <event-type> <handler-fn>   # Subscribe to event
events-unsubscribe <handler-id>              # Unsubscribe
events-emit <event-type> <data>              # Emit event (for testing)
events-parse-bspc <line>                     # Parse bspc subscribe line
events-filter <pattern>                      # Filter events by pattern
events-debounce <interval>                   # Set debounce interval
events-batch <interval>                      # Batch events for processing
```

**Integration Pattern**:
```zsh
# In daemon main
source $(which _events)

# Features subscribe to events
events-subscribe "node_add" balance_handle_event
events-subscribe "node_remove" balance_handle_event
events-subscribe "node_add" ventilate_handle_inhale
events-subscribe "node_remove" ventilate_handle_exhale

# Main event loop
bspc subscribe node_add node_remove node_state node_flag node_geometry \
              desktop_add desktop_remove desktop_focus \
              monitor_add monitor_remove monitor_focus | \
while read -r event_line; do
    event_type=$(events-parse-bspc "$event_line" type)
    events-emit "$event_type" "$event_line"
done
```

---

#### _config (May need to create or enhance)
**Purpose**: Configuration management (JSON loading, validation, defaults)

**Why Needed**: Unified configuration, validation, runtime updates

**Functions Needed**:
```zsh
config-load <package-name>                 # Load config from standard location
config-get <key> [default]                 # Get config value (dot notation)
config-set <key> <value>                   # Set config value
config-validate [schema-file]              # Validate config against schema
config-reload                              # Reload config from disk
config-path                                # Get config file path
config-exists <key>                        # Check if key exists
config-merge <file>                        # Merge config from file
```

**Integration Pattern**:
```zsh
# In daemon main
source $(which _config)

# Load config
config-load "bsp"

# Check if feature enabled
if [[ "$(config-get 'features.balance.enabled' 'false')" == "true" ]]; then
    lifecycle-start "balance"
fi

# Get feature-specific config
border_width=$(config-get 'features.flag.marked.border_width' '3')
```

**Configuration Schema**:
```json
{
  "version": "2.0",
  "features": {
    "balance": {
      "enabled": true,
      "strategy": "equal",
      "exclude_classes": []
    },
    "desktops": {
      "enabled": true,
      "start_id": 1
    },
    "flag": {
      "enabled": true,
      "marked": {
        "border_width": 3,
        "border_color": "#0000ff"
      },
      "floating": {
        "border_width": 2,
        "border_color": "#00ff00"
      },
      "hidden": {},
      "sticky": {},
      "locked": {},
      "private": {}
    },
    "floating_borders": {
      "enabled": true,
      "border_width": 0
    },
    "ventilate": {
      "enabled": true,
      "terminals": ["Alacritty", "terminal-drop", "Code"],
      "no_inhale": ["Rofi", "xev", "xprop", "gcr-prompter", "firefox", "Google-chrome"]
    }
  },
  "ipc": {
    "socket": "${XDG_RUNTIME_DIR}/bsp.sock",
    "timeout": 5
  },
  "logging": {
    "level": "INFO",
    "file": "${XDG_STATE_HOME}/bsp/bsp.log",
    "max_size": "10M",
    "rotate": 5
  },
  "layouts": {
    "directory": "${XDG_CONFIG_HOME}/bsp/layouts"
  }
}
```

---

#### _cache (May need to create or enhance)
**Purpose**: Cache bspwm query results for performance

**Why Needed**: Reduce bspc calls, improve event handling speed

**Functions Needed**:
```zsh
cache-get <key>                           # Get cached value
cache-set <key> <value> <ttl>             # Set cached value with TTL
cache-invalidate <pattern>                # Invalidate cache by pattern
cache-clear                               # Clear all cache
cache-stats                               # Show cache statistics
```

**Integration Pattern**:
```zsh
# In _bspwm library
source $(which _cache)

bspwm-query-nodes() {
    cache_key="bspwm:nodes:$1"
    cached=$(cache-get "$cache_key")

    if [[ -n "$cached" ]]; then
        echo "$cached"
        return
    fi

    result=$(bspc query -N "$@")
    cache-set "$cache_key" "$result" 5  # 5 second TTL
    echo "$result"
}

# Invalidate on events
events-subscribe "node_add" "cache-invalidate 'bspwm:nodes:*'"
events-subscribe "node_remove" "cache-invalidate 'bspwm:nodes:*'"
```

---

### bspwm Abstraction (NEW LIBRARY - HIGH PRIORITY)

#### _bspwm (NEW - Must be created)
**Purpose**: Complete bspwm/bspc abstraction layer

**Why Needed**:
- Centralized bspwm interaction
- Error handling for all bspc commands
- Caching for performance
- Testing (mock bspc commands)
- Consistent interface

**Functions to Implement**:

##### Query Functions
```zsh
bspwm-query-nodes [selector]          # Get node IDs
bspwm-query-desktops [selector]       # Get desktop IDs
bspwm-query-monitors [selector]       # Get monitor IDs
bspwm-query-focused-node              # Get focused node ID
bspwm-query-focused-desktop           # Get focused desktop ID
bspwm-query-focused-monitor           # Get focused monitor ID
bspwm-query-tree [desktop]            # Get tree structure (JSON)
```

##### Node Manipulation
```zsh
bspwm-node-set-state <node> <state>           # Set node state (tiled, floating, fullscreen)
bspwm-node-set-flag <node> <flag> <value>     # Set node flag (on/off)
bspwm-node-set-border <node> <width>          # Set border width
bspwm-node-set-color <node> <color>           # Set border color
bspwm-node-move <node> <desktop>              # Move to desktop
bspwm-node-swap <node1> <node2>               # Swap nodes
bspwm-node-focus <node>                       # Focus node
bspwm-node-close <node>                       # Close node
bspwm-node-kill <node>                        # Kill node
bspwm-node-balance <node>                     # Balance subtree
bspwm-node-preselect <node> <direction> <ratio>  # Preselect split
```

##### Desktop Manipulation
```zsh
bspwm-desktop-add <monitor> <name>            # Add desktop
bspwm-desktop-remove <desktop>                # Remove desktop
bspwm-desktop-rename <desktop> <name>         # Rename desktop
bspwm-desktop-focus <desktop>                 # Focus desktop
bspwm-desktop-swap <desktop1> <desktop2>      # Swap desktops
```

##### Monitor Manipulation
```zsh
bspwm-monitor-add <name>                      # Add monitor
bspwm-monitor-remove <monitor>                # Remove monitor
bspwm-monitor-focus <monitor>                 # Focus monitor
bspwm-monitor-rename <monitor> <name>         # Rename monitor
```

##### Configuration
```zsh
bspwm-config-get <key>                        # Get config value
bspwm-config-set <key> <value>                # Set config value
bspwm-config-set-node <node> <key> <value>    # Set node-specific config
```

##### Event Subscription
```zsh
bspwm-subscribe <events...>                   # Subscribe to events (wrapper)
```

##### Window Properties (via xprop)
```zsh
bspwm-window-class <window>                   # Get WM_CLASS
bspwm-window-name <window>                    # Get WM_NAME
bspwm-window-command <window>                 # Get WM_COMMAND
bspwm-window-pid <window>                     # Get _NET_WM_PID
```

##### Utility Functions
```zsh
bspwm-is-running                              # Check if bspwm is running
bspwm-version                                 # Get bspwm version
bspwm-validate-selector <selector>            # Validate selector syntax
```

**Implementation Details**:
- Wrap all bspc calls with error handling
- Return status codes (0=success, 1=error)
- Log all commands (debug level)
- Cache query results (with event-based invalidation)
- Validate inputs (selectors, node IDs)
- Handle missing dependencies (xprop)

**Integration Pattern**:
```zsh
# In features
source $(which _bspwm)

# Instead of: bspc query -N -d -n .window
nodes=$(bspwm-query-nodes "-d -n .window")

# Instead of: bspc node $node -B
bspwm-node-balance "$node"

# Instead of: xprop -id $window WM_CLASS
class=$(bspwm-window-class "$window")
```

**Dependencies**:
- _cache: For query caching
- _log: For command logging
- _events: For cache invalidation

**Reusability**: HIGH - Any bspwm utility in dotfiles would benefit

---

### Optional Dependencies (Enhanced Features)

#### _rofi (May exist or need to create)
**Purpose**: Interactive Rofi menus

**Why Needed**: Layout selector, desktop switcher, feature toggler

**Functions Needed**:
```zsh
rofi-menu <options>                    # Show menu with options
rofi-select <prompt> <items>           # Selection dialog
rofi-input <prompt>                    # Text input dialog
rofi-confirm <prompt>                  # Confirmation dialog
```

**Integration Pattern**:
```zsh
# In bsp-layout (select subcommand)
source $(which _rofi)

layouts=$(bsp layout list --names-only)
selected=$(rofi-select "Select layout:" "$layouts")

[[ -n "$selected" ]] && bsp layout apply "$selected"
```

**Use Cases**:
- `bsp layout select`: Choose layout interactively
- `bsp workspace select`: Choose workspace interactively
- `bsp feature toggle`: Enable/disable features interactively

---

#### _ui (May exist or need to create)
**Purpose**: Terminal UI formatting

**Why Needed**: Pretty-print status, tree visualization, tables

**Functions Needed**:
```zsh
ui-table <data>                        # Format table
ui-tree <data>                         # Format tree (ASCII art)
ui-status <items>                      # Status indicators (✓, ✗, ●)
ui-progress <percent>                  # Progress bar
ui-color <color> <text>                # Colored text
ui-bold <text>                         # Bold text
ui-header <text>                       # Header with separator
```

**Integration Pattern**:
```zsh
# In bsp-status
source $(which _ui)

ui-header "BSP Window Manager Utilities"
echo ""
ui-status "Daemon" "RUNNING (PID 12345)"
ui-status "Uptime" "2 hours 34 minutes"
echo ""
ui-header "Features"
ui-status "balance" "ENABLED" "success"
ui-status "desktops" "ENABLED" "success"
```

**Use Cases**:
- `bsp status`: Formatted status display
- `bsp tree`: ASCII art tree visualization
- Error messages: Colored and formatted

---

#### _notify (May exist or need to create)
**Purpose**: Desktop notifications

**Why Needed**: Notify on desktop changes, layout changes, errors

**Functions Needed**:
```zsh
notify-send <title> <message> [urgency]   # Send notification
notify-error <message>                    # Error notification (critical)
notify-warn <message>                     # Warning notification
notify-info <message>                     # Info notification
```

**Integration Pattern**:
```zsh
# In features/desktops.zsh
source $(which _notify)

# When desktop added
notify-info "Desktop Added" "Desktop $new_id created"

# When error occurs
notify-error "BSP Error" "Failed to balance windows"
```

**Use Cases**:
- Desktop created/removed notifications
- Layout applied notifications
- Error notifications (always)

---

### Layout Management (NEW LIBRARY - MEDIUM PRIORITY)

#### _layout (NEW - Should be created)
**Purpose**: Layout profile management

**Why Needed**: Save/restore window layouts, reproducible workspaces

**Functions to Implement**:
```zsh
layout-list                               # List available layouts
layout-save <name>                        # Save current layout
layout-load <name>                        # Load layout (return JSON)
layout-apply <desktop> <layout-json>      # Apply layout to desktop
layout-delete <name>                      # Delete layout
layout-export <name> <file>               # Export layout to file
layout-import <file>                      # Import layout from file
layout-validate <layout-json>             # Validate layout structure
```

**Layout JSON Schema**:
```json
{
  "name": "dev",
  "description": "Development layout with editor and terminals",
  "version": "1.0",
  "created": "2025-11-07T12:34:56Z",
  "tree": {
    "type": "internal",
    "split": "vertical",
    "ratio": 0.618,
    "first": {
      "type": "window",
      "class": "Code",
      "title": null,
      "state": "tiled"
    },
    "second": {
      "type": "internal",
      "split": "horizontal",
      "ratio": 0.5,
      "first": {
        "type": "window",
        "class": "Alacritty",
        "title": null,
        "state": "tiled"
      },
      "second": {
        "type": "window",
        "class": "Alacritty",
        "title": null,
        "state": "tiled"
      }
    }
  }
}
```

**Implementation Details**:
- Capture current desktop tree structure
- Store as JSON in ~/.config/bsp/layouts/
- Apply by creating preselections and receptacles
- Optionally auto-launch applications
- Handle missing applications gracefully

**Integration Pattern**:
```zsh
# In bsp-layout
source $(which _layout)
source $(which _bspwm)

# Save layout
layout-save "my-layout"

# Apply layout
layout_json=$(layout-load "my-layout")
focused_desktop=$(bspwm-query-focused-desktop)
layout-apply "$focused_desktop" "$layout_json"
```

**Dependencies**:
- _bspwm: Query tree, create preselections
- _config: Get layouts directory path

**Reusability**: MEDIUM - Useful for other bspwm layout tools

---

### Function Mapping: Commands → Libraries

**Command**: `bsp daemon start`
- _singleton: Check if already running
- _lifecycle: Start daemon process
- _log: Log startup

**Command**: `bsp status`
- IPC: Query daemon for feature status
- _bspwm: Query windows, desktops, monitors
- _config: Read config file
- _ui: Format output

**Command**: `bsp balance on`
- IPC: Send enable message to daemon
- _lifecycle: Start balance feature in daemon
- _events: Subscribe to events in daemon
- _config: Update config (persist state)

**Command**: `bsp side west`
- _bspwm: Query focused node
- _bspwm: Manipulate node (focus, set split, set state)
- _log: Log actions

**Command**: `bsp shroud cover`
- Bash: Call polybar-msg (external)
- _bspwm: Query leaf nodes
- _bspwm: Set hidden flag

**Command**: `bsp layout save my-layout`
- _layout: Capture tree structure
- _bspwm: Query tree
- _config: Get layouts directory
- File I/O: Write JSON file

**Command**: `bsp tree`
- _bspwm: Query tree structure
- _ui: Format ASCII art tree

**Command**: `bsp config edit`
- _config: Get config file path
- Bash: Open $EDITOR
- _config: Validate after edit
- IPC: Reload daemon config

**Feature**: `balance` (in daemon)
- _events: Subscribe to node_add, node_remove, etc.
- _bspwm: Query nodes
- _bspwm: Balance nodes
- _cache: Cache query results
- _log: Log events and actions

**Feature**: `ventilate` (in daemon)
- _events: Subscribe to node_add, node_remove
- _bspwm: Query windows
- _bspwm: Set hidden flag
- _config: Get terminals and no_inhale lists
- _string: Lowercase comparison
- File I/O: Read/write inhaled state
- _log: Log swallow/exhale actions

---

### Event Emission Strategy

**Event Bus Pattern**:
- Single `bspc subscribe` in daemon
- Parse event lines into structured events
- Route to subscribed handlers
- Allow multiple handlers per event type

**Event Types**:
```
node_add           → balance, desktops, ventilate
node_remove        → balance, desktops, ventilate
node_state         → balance, desktops, floating-borders
node_flag          → flag
node_geometry      → balance
desktop_add        → (future: notifications)
desktop_remove     → (future: notifications)
desktop_focus      → (future: status updates)
monitor_add        → (future: layout profiles)
monitor_remove     → (future: layout profiles)
monitor_focus      → (future: status updates)
```

**Debouncing Strategy**:
- `node_add`, `node_remove`: No debounce (immediate action needed)
- `node_geometry`: Debounce 100ms (avoid balance spam during resize)
- `node_state`: Debounce 50ms (avoid flicker during state changes)

**Event Emission in Features**:
```zsh
# In balance feature
events-subscribe "node_add" balance_handle_event
events-subscribe "node_remove" balance_handle_event
events-subscribe "node_state" balance_handle_event
events-subscribe "node_geometry" balance_handle_event

balance_handle_event() {
    local event_line="$1"

    log-debug "[balance] Event: ${event_line}"

    # Balance logic
    for window in $(bspwm-query-nodes ".window"); do
        bspwm-node-balance "${window}#@north"
    done
}
```

---

### Cache Usage Patterns

**What to Cache**:
- bspc query results (nodes, desktops, monitors) - TTL: 5 seconds
- Tree structure - TTL: 5 seconds
- Window properties (class, name) - TTL: 60 seconds
- Config values (rarely change) - TTL: 3600 seconds

**When to Invalidate**:
- `node_add`, `node_remove`: Invalidate node queries
- `desktop_add`, `desktop_remove`: Invalidate desktop queries
- `monitor_add`, `monitor_remove`: Invalidate monitor queries
- Config reload: Invalidate all config cache

**Cache Keys**:
```
bspwm:nodes:<selector>
bspwm:desktops:<selector>
bspwm:monitors:<selector>
bspwm:tree:<desktop>
bspwm:window:<id>:class
bspwm:window:<id>:name
config:<key>
```

---

### Configuration Approach

**Single Config File**: `~/.config/bsp/config.json`

**Validation**: JSON schema validation on load and update

**Runtime Updates**: IPC command to reload config without daemon restart

**Backward Compatibility**: Migrate old text files (no_inhale, terminals) to JSON on first run

**Default Values**: Embedded in code, config file only overrides

**Profile Support**: Multiple config files, switch with `--config` flag

---

## Configuration Schema

### Main Configuration

**File**: `~/.config/bsp/config.json`

```json
{
  "$schema": "https://raw.githubusercontent.com/andronics/dotfiles/main/bsp/config.schema.json",
  "version": "2.0",

  "features": {
    "balance": {
      "enabled": true,
      "strategy": "equal",
      "exclude_classes": [],
      "events": ["node_add", "node_remove", "node_state", "node_geometry"]
    },

    "desktops": {
      "enabled": true,
      "start_id": 1,
      "auto_naming": false,
      "naming_pattern": "Desktop {id}",
      "events": ["node_add", "node_remove", "node_state", "node_geometry"]
    },

    "flag": {
      "enabled": true,
      "events": ["node_flag"],
      "marked": {
        "border_width": 3,
        "border_color": "#0000ff"
      },
      "floating": {
        "border_width": 2,
        "border_color": "#00ff00"
      },
      "hidden": {
        "border_width": 0
      },
      "sticky": {
        "border_color": "#ffff00"
      },
      "locked": {
        "border_color": "#ff0000"
      },
      "private": {
        "border_color": "#ff00ff"
      }
    },

    "floating_borders": {
      "enabled": true,
      "border_width": 0,
      "events": ["node_state"]
    },

    "ventilate": {
      "enabled": true,
      "terminals": [
        "Alacritty",
        "terminal-drop",
        "Code"
      ],
      "no_inhale": [
        "Rofi",
        "xev",
        "xprop",
        "gcr-prompter",
        "firefox",
        "Google-chrome"
      ],
      "events": ["node_add", "node_remove"],
      "state_file": "${XDG_STATE_HOME}/bsp/inhaled.json"
    }
  },

  "ipc": {
    "socket": "${XDG_RUNTIME_DIR}/bsp.sock",
    "timeout": 5,
    "buffer_size": 4096
  },

  "logging": {
    "level": "INFO",
    "file": "${XDG_STATE_HOME}/bsp/bsp.log",
    "max_size": "10M",
    "rotate": 5,
    "format": "[{timestamp}] [{level}] [{context}] {message}"
  },

  "cache": {
    "enabled": true,
    "ttl": {
      "nodes": 5,
      "desktops": 5,
      "monitors": 5,
      "tree": 5,
      "window_properties": 60,
      "config": 3600
    },
    "file": "${XDG_CACHE_HOME}/bsp/cache.json"
  },

  "layouts": {
    "directory": "${XDG_CONFIG_HOME}/bsp/layouts",
    "auto_apply": false,
    "default_layout": "default"
  },

  "notifications": {
    "enabled": true,
    "events": {
      "desktop_added": true,
      "desktop_removed": true,
      "layout_applied": true,
      "feature_enabled": false,
      "feature_disabled": false,
      "errors": true
    },
    "urgency": {
      "info": "normal",
      "warning": "normal",
      "error": "critical"
    }
  },

  "ui": {
    "colors": true,
    "unicode": true,
    "tree": {
      "symbols": {
        "vertical": "│",
        "horizontal": "─",
        "corner": "└",
        "tee": "├"
      }
    }
  }
}
```

### Configuration Validation

**JSON Schema**: Define schema for validation

**Validation Rules**:
- All required keys present
- Types correct (boolean, number, string, array)
- Enum values valid (log levels, strategies)
- Paths exist or can be created
- No unknown keys (warn about typos)

**Migration**: Auto-migrate old config versions

**Example Validation Output**:
```bash
$ bsp config validate
Validating configuration...
✓ Syntax valid (JSON)
✓ Schema valid
✓ All required keys present
! Warning: Unknown key 'features.balance.unknown_option' (ignored)
✓ All paths valid
Configuration is valid (1 warning)
```

---

## UI/UX Design

### CLI Output Formats

#### Success Messages
```bash
$ bsp balance on
Feature 'balance' enabled
```

#### Error Messages
```bash
$ bsp balance on
Error: Daemon not running
Try: bsp daemon start

$ bsp layout apply nonexistent
Error: Layout 'nonexistent' not found
Available layouts: default, dev, media, work
```

#### Status Display
```bash
$ bsp status

BSP Window Manager Utilities
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Daemon: RUNNING (PID 12345)
Uptime: 2 hours 34 minutes
Socket: /run/user/1000/bsp.sock

Features:
  ✓ balance           ENABLED
  ✓ desktops          ENABLED  (start_id: 1)
  ✓ flag              ENABLED
  ✓ floating-borders  ENABLED
  ✓ ventilate         ENABLED  (3 terminals, 6 no-inhale)

Window Manager:
  Windows: 12
  Desktops: 5 (2 occupied, 3 empty)
  Monitors: 2 (eDP-1, HDMI-1)
  Focused: Desktop 1, Window 0x00400001

Layouts:
  - default
  - dev
  - media
  - work

Configuration:
  File: /home/andronics/.config/bsp/config.json
  Last modified: 2 hours ago
  Valid: ✓
```

#### Tree Visualization
```bash
$ bsp tree --focused

Desktop 1 (focused)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├─ [V] 0x00400001 (50%)
│  ├─ [T] Code - ~/project/main.rs
│  └─ [H] 0x00400002 (50%)
│     ├─ [T] Alacritty - zsh
│     └─ [T] Alacritty - vim
└─ [F] firefox - Mozilla Firefox

Legend:
  [V] Vertical split   [H] Horizontal split
  [T] Tiled            [F] Floating
  [*] Fullscreen       [M] Marked
```

#### JSON Output
```bash
$ bsp status --json
{
  "daemon": {
    "status": "running",
    "pid": 12345,
    "uptime": 9240,
    "socket": "/run/user/1000/bsp.sock"
  },
  "features": {
    "balance": {"enabled": true},
    "desktops": {"enabled": true, "config": {"start_id": 1}},
    "flag": {"enabled": true},
    "floating_borders": {"enabled": true},
    "ventilate": {
      "enabled": true,
      "config": {
        "terminals_count": 3,
        "no_inhale_count": 6
      }
    }
  },
  "window_manager": {
    "windows": 12,
    "desktops": 5,
    "desktops_occupied": 2,
    "monitors": 2,
    "focused_desktop": 1,
    "focused_window": "0x00400001"
  },
  "layouts": ["default", "dev", "media", "work"],
  "config": {
    "file": "/home/andronics/.config/bsp/config.json",
    "last_modified": 7200,
    "valid": true
  }
}
```

### Rofi Integration

#### Layout Selector
```bash
$ bsp layout select
```

**Rofi Menu**:
```
Select Layout
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
› default     - Default balanced layout
  dev         - Development layout (editor + terminals)
  media       - Media layout (player + controls)
  work        - Work layout (browser + apps)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Implementation**:
```zsh
# In bsp-layout (select subcommand)
source $(which _rofi)
source $(which _layout)

# Get layout list with descriptions
layouts=$(layout-list --with-descriptions)

# Show Rofi menu
selected=$(echo "$layouts" | rofi-menu -p "Select Layout" -format "name")

# Apply selected layout
[[ -n "$selected" ]] && layout-apply "$(bspwm-query-focused-desktop)" "$selected"
```

#### Feature Toggle Menu
```bash
$ bsp feature toggle
```

**Rofi Menu**:
```
Toggle Features
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ balance           - Auto-balance window tree
✓ desktops          - Dynamic desktop management
✓ flag              - React to window flag changes
✓ floating-borders  - Remove borders from floating windows
✓ ventilate         - Terminal window swallowing
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Notification Design

#### Desktop Events
```
Title: Desktop Added
Body: Desktop 6 created
Icon: window-new
Urgency: normal
```

```
Title: Desktop Removed
Body: Desktop 6 removed (empty)
Icon: window-close
Urgency: normal
```

#### Layout Events
```
Title: Layout Applied
Body: Applied 'dev' layout to Desktop 1
Icon: window-tile
Urgency: normal
```

#### Error Notifications
```
Title: BSP Error
Body: Failed to balance windows: bspwm not running
Icon: error
Urgency: critical
```

**Implementation**:
```zsh
# In features/desktops.zsh
source $(which _notify)

# When desktop added
notify-info "Desktop Added" "Desktop ${desktop_id} created"

# When desktop removed
notify-info "Desktop Removed" "Desktop ${desktop_id} removed (empty)"

# In bsp-layout
notify-info "Layout Applied" "Applied '${layout_name}' layout to Desktop ${desktop_id}"

# On error
notify-error "BSP Error" "Failed to balance windows: ${error_message}"
```

### Interactive Modes

#### Watch Mode
```bash
$ bsp status --watch
```

Continuously updates status display (1 second refresh)

Press `q` to quit

#### TUI Mode (Future)
```bash
$ bsp tui
```

Interactive terminal UI with:
- Live status dashboard
- Feature toggles
- Tree visualization
- Log viewer
- Config editor

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1 - 7 days)

**Goal**: Core infrastructure, _bspwm library, basic daemon with one feature

**Tasks**:

#### Day 1-3: Create _bspwm Library
- Implement query wrappers (bspwm-query-nodes, -desktops, -monitors)
- Implement node manipulation (bspwm-node-set-state, -set-flag, -balance)
- Implement desktop manipulation (bspwm-desktop-add, -remove, -focus)
- Implement window properties (bspwm-window-class, -name, -command)
- Add error handling to all functions
- Add caching to query functions
- Write tests (mock bspc commands)

**Deliverables**:
- ~/.local/bin/lib/_bspwm (complete library)
- Test suite for _bspwm
- Documentation for all functions

#### Day 4: Create Main Dispatcher
- Implement ~/.pkgs/bsp/.local/bin/bsp (main dispatcher)
- Argument parsing with _args
- Subcommand routing
- Help system (bsp-help)
- Version info (bsp-version)
- Global flags (--help, --version, --debug, --config)

**Deliverables**:
- Main bsp dispatcher
- bsp-help command
- bsp-version command

#### Day 5-6: Create Daemon Foundation
- Implement ~/.pkgs/bsp/.local/libexec/bsp/bspd (main daemon)
- Event loop with bspc subscribe
- Event parsing and routing (_events library)
- Feature loading system
- Lifecycle management (_lifecycle library)
- Configuration loading (_config library)
- IPC server (Unix socket) - basic implementation
- Singleton enforcement (_singleton)
- Cleanup handlers (_onexit or lifecycle)

**Deliverables**:
- bspd daemon (working event loop)
- Event routing to features
- Feature loading infrastructure
- IPC server (basic)

#### Day 7: Migrate One Feature (Balance)
- Port bsp-balance to features/balance.zsh
- Implement as feature module (balance_start, balance_stop, balance_handle_event)
- Subscribe to events (node_add, node_remove, node_state, node_geometry)
- Use _bspwm library for all bspwm interaction
- Test event handling
- Verify functionality matches original

**Deliverables**:
- features/balance.zsh (complete)
- Working balance feature in daemon
- bsp-feature command (enable/disable balance)

**Success Criteria**:
- [ ] `bsp --help` shows help
- [ ] `bsp --version` shows version
- [ ] `bsp daemon start` starts daemon
- [ ] Daemon logs to standard location
- [ ] `bsp balance on` enables balance via IPC
- [ ] Balance feature works exactly as before
- [ ] _bspwm library tests pass

---

### Phase 2: Feature Migration (Week 2 - 7 days)

**Goal**: Migrate all existing features to new architecture

**Tasks**:

#### Day 1: Migrate Desktops Feature
- Port bsp-desktops to features/desktops.zsh
- Implement start_id configuration
- Subscribe to events (node_add, node_remove, node_state, node_geometry)
- Dynamic desktop add/remove logic
- Test functionality

**Deliverables**:
- features/desktops.zsh (complete)
- `bsp desktops on/off/status` working
- Config support for start_id

#### Day 2: Migrate Flag Feature
- Port bsp-flag to features/flag.zsh
- **COMPLETE implementation** (all 6 flag types, not just marked)
- Subscribe to node_flag events
- Configuration for each flag type (borders, colors)
- Test all flag types (marked, floating, hidden, sticky, locked, private)

**Deliverables**:
- features/flag.zsh (complete, all flags implemented)
- `bsp flag on/off/status/config` working
- All 6 flag handlers functional

#### Day 3: Migrate Floating Borders Feature
- Port bsp-floating-borders to features/floating-borders.zsh
- Subscribe to node_state events
- Simple border removal logic
- Test functionality

**Deliverables**:
- features/floating-borders.zsh (complete)
- `bsp floating-borders on/off/status` working

#### Day 4: Migrate Ventilate Feature
- Port bsp-ventilate to features/ventilate.zsh
- Migrate configuration to JSON (terminals, no_inhale)
- Subscribe to events (node_add, node_remove)
- Inhale/exhale logic
- State tracking (migrate from text file to JSON)
- Test swallowing behavior

**Deliverables**:
- features/ventilate.zsh (complete)
- `bsp ventilate on/off/status/config` working
- Config migration from text files to JSON
- State tracking in JSON format

#### Day 5: Migrate CLI Utilities
- Port bsp-side to libexec/bsp/bsp-side
- Port bsp-shroud to libexec/bsp/bsp-shroud
- Port bsp-fullscreen to libexec/bsp/bsp-fullscreen
- All use _bspwm library
- Test exact behavior match

**Deliverables**:
- bsp-side command (working)
- bsp-shroud command (working)
- bsp-fullscreen command (working)
- All use _bspwm library

#### Day 6-7: Configuration System
- Create comprehensive config.json schema
- Implement config validation (_config library enhancement)
- Implement config migration script (old → new)
- Implement default value system
- Test config loading, validation, migration
- Implement `bsp config` commands (get, set, edit, validate, path, reload)

**Deliverables**:
- config.json schema (complete)
- Config validation working
- Config migration script
- `bsp config` commands working

**Success Criteria**:
- [ ] All 8 utilities work exactly as before
- [ ] Single daemon replaces 5 separate daemons
- [ ] Config loaded from JSON
- [ ] Config migration working
- [ ] Features can be toggled at runtime via IPC
- [ ] Backward compatibility maintained (old command names work)

---

### Phase 3: Enhanced Features (Week 3 - 7 days)

**Goal**: Add IPC, status, tree, improved features, error handling

**Tasks**:

#### Day 1-2: IPC System
- Implement complete Unix socket server in daemon
- Implement IPC client in CLI commands
- Define IPC protocol (JSON messages)
- Commands: enable/disable feature, get status, reload config
- Response serialization (JSON)
- Timeout and retry logic
- Test IPC communication

**Deliverables**:
- Complete IPC server in daemon
- IPC client library
- IPC protocol defined
- All feature control via IPC working

#### Day 3: Status and Monitoring
- Implement bsp-status command (comprehensive status)
- Human-readable output format
- JSON output format (--json flag)
- Watch mode (--watch flag)
- Feature status display
- Window manager status (windows, desktops, monitors)
- Daemon health check
- Test status reporting

**Deliverables**:
- bsp-status command (complete)
- Human-readable and JSON output
- Watch mode working

#### Day 4: Tree Visualization
- Implement bsp-tree command
- Query tree structure via _bspwm
- ASCII art tree formatting via _ui
- Color coding by state (tiled, floating, fullscreen, marked)
- Options: --focused, --desktop, --color, --json
- Test tree display

**Deliverables**:
- bsp-tree command (complete)
- ASCII art visualization working
- JSON output working

#### Day 5: Improve Existing Features
- **Flag**: Complete all 6 flag handlers (marked, floating, hidden, sticky, locked, private)
- **Balance**: Smart balance (heuristics, configurable strategies)
- **Balance**: Exclude classes from balancing
- **Desktops**: Desktop auto-naming option
- **Ventilate**: Better class detection (handle edge cases)
- Test all improvements

**Deliverables**:
- All flag handlers implemented and tested
- Smart balance working
- Desktop auto-naming working
- Better ventilate detection

#### Day 6: Error Handling
- Graceful degradation (bspwm not running)
- Error recovery in daemon (feature crashes don't kill daemon)
- Better error messages (helpful, actionable)
- Input validation (selectors, node IDs, config values)
- Daemon health monitoring (auto-restart failed features)
- Test error scenarios

**Deliverables**:
- Robust error handling throughout
- Graceful degradation working
- Helpful error messages
- Health monitoring working

#### Day 7: Testing and Bug Fixes
- Integration tests for all features
- End-to-end tests for workflows
- Bug fixes from testing
- Performance testing (event handling latency)
- Code cleanup

**Deliverables**:
- Test suite (comprehensive)
- All tests passing
- No known bugs

**Success Criteria**:
- [ ] `bsp status` shows accurate, comprehensive status
- [ ] `bsp tree` shows beautiful ASCII art tree
- [ ] All flag handlers implemented and working
- [ ] Features handle errors gracefully
- [ ] Daemon survives bspwm restart
- [ ] Feature crashes don't kill daemon
- [ ] Error messages are helpful

---

### Phase 4: Advanced Features (Week 4 - 7 days)

**Goal**: Layout management, Rofi integration, documentation, polish

**Tasks**:

#### Day 1-2: Create _layout Library
- Implement layout schema (JSON)
- Implement layout-save (capture tree structure)
- Implement layout-load (read from file)
- Implement layout-apply (create preselections, apply)
- Implement layout-validate (validate structure)
- Implement layout-export/import
- Test layout operations
- Create default layouts (default, dev, media, work)

**Deliverables**:
- ~/.local/bin/lib/_layout (complete library)
- Layout schema defined
- Default layouts created

#### Day 3: Layout Commands
- Implement bsp-layout command
- Subcommands: list, save, load, apply, delete, export, import, select
- Interactive Rofi selector (select subcommand)
- Test all layout commands
- Test layout application

**Deliverables**:
- bsp-layout command (complete)
- All subcommands working
- Rofi integration working

#### Day 4: Rofi Integration
- Implement _rofi library integration
- Layout selector (bsp layout select)
- Desktop switcher (future: bsp desktop select)
- Feature toggler (future: bsp feature toggle)
- Test Rofi menus

**Deliverables**:
- Rofi integration working
- Layout selector beautiful and functional

#### Day 5-6: Documentation
- README.md with quick start, installation, features
- Man pages for all commands (bsp.1, bsp-balance.1, etc.)
- Configuration guide with examples
- Architecture documentation (this document, refined)
- Migration guide (old → new)
- Example workflows
- Troubleshooting guide

**Deliverables**:
- README.md (comprehensive)
- Man pages (complete)
- Configuration guide
- Migration guide

#### Day 7: Testing and Polish
- Final integration tests
- Performance testing and optimization
- UI polish (colors, formatting)
- Bug fixes
- Code cleanup
- Release preparation

**Deliverables**:
- Polished, production-ready code
- All tests passing
- Documentation complete
- No known bugs

**Success Criteria**:
- [ ] Layouts can be saved, loaded, applied
- [ ] Rofi menus work beautifully
- [ ] Documentation is comprehensive and helpful
- [ ] Tests passing
- [ ] No known bugs
- [ ] Code is clean and maintainable
- [ ] Ready for users

---

### Phase 5: Polish & Release (Bonus Week - 5 days)

**Goal**: Production-ready release with systemd, migration, optimizations

**Tasks**:

#### Day 1: Systemd Integration
- Create bsp.service (single service replacing 5)
- Auto-start on login (optional)
- Proper shutdown handling
- Service status integration with `bsp daemon status`
- Test systemd service

**Deliverables**:
- bsp.service (complete)
- Systemd integration working

#### Day 2: Migration Guide and Script
- Migration script (disable old services, enable new service)
- Config migration (text files → JSON)
- Backward compatibility shims (bsp-balance → bsp balance)
- Deprecation notices
- Breaking changes documentation
- Test migration on fresh system

**Deliverables**:
- Migration script (complete)
- Backward compatibility shims
- Migration documentation

#### Day 3: Performance Optimization
- Event debouncing implementation
- Query caching optimization
- Reduce bspc calls (use cache more)
- Profile hot paths
- Benchmark and measure improvements
- Test performance

**Deliverables**:
- Optimized event handling
- Improved cache usage
- Performance benchmarks

#### Day 4: UI Polish
- Better status display (formatting, colors)
- Progress indicators for long operations
- Tab completion (zsh and bash)
- Color themes
- Test UI

**Deliverables**:
- Polished UI
- Tab completion working

#### Day 5: Release Preparation
- Final testing
- Release notes
- Changelog
- Version tagging
- Release artifacts
- Announcement

**Deliverables**:
- Release v2.0.0
- Release notes
- Changelog

**Success Criteria**:
- [ ] Systemd service works perfectly
- [ ] Migration from old version is smooth
- [ ] Performance is excellent
- [ ] UI is polished and beautiful
- [ ] Ready for production use

---

### Effort Estimate

**Total**: **4-5 weeks** (28-35 days)

**Breakdown**:
- Phase 1 (Foundation): 7 days
- Phase 2 (Migration): 7 days
- Phase 3 (Enhancement): 7 days
- Phase 4 (Advanced): 7 days
- Phase 5 (Polish): 5 days (optional/bonus)

**Confidence Level**: **75% (High)**

**Variables**:
- _bspwm library complexity (might take 1-2 extra days)
- IPC implementation (depends on chosen mechanism, might take 1 extra day)
- Testing complexity (requires X11 environment, might take extra time)
- Feature scope creep (many exciting features possible)

**Assumptions**:
- Full-time work (8 hours/day, 40 hours/week)
- Existing libraries (_events, _lifecycle, _config) are functional or easy to create/enhance
- bspwm environment available for testing
- No major blockers or unknowns

---

## Risk Mitigation

### Technical Risks

#### Risk 1: Event Handling Complexity
**Likelihood**: Medium | **Impact**: High

**Description**: Coordinating multiple features reacting to same events could cause race conditions or unexpected interactions.

**Mitigation**:
- Use event bus pattern with clear handler ordering
- Features subscribe independently, don't share mutable state
- Implement event debouncing/throttling to reduce event spam
- Extensive testing with multiple features enabled simultaneously
- Add event logging for debugging (see exact order of events and handlers)
- If race conditions occur, implement handler priorities

---

#### Risk 2: bspwm Dependency
**Likelihood**: Low | **Impact**: High

**Description**: bspwm API changes or bugs could break functionality.

**Mitigation**:
- Abstract ALL bspwm interaction through _bspwm library
- Version detection (check bspwm version on start)
- Compatibility checks (test for required bspc commands)
- Graceful degradation on API changes (fallback to safe defaults)
- Comprehensive error handling (catch bspc failures)
- Monitor bspwm development (follow upstream changes)

---

#### Risk 3: IPC Complexity
**Likelihood**: Medium | **Impact**: Medium

**Description**: Implementing reliable IPC between CLI and daemon is non-trivial.

**Mitigation**:
- Use simple Unix sockets (not D-Bus initially, avoid complexity)
- Implement timeout and retry logic (5 second timeout, 3 retries)
- Fall back to direct mode if daemon unreachable (for CLI utilities)
- Extensive IPC testing (stress test, concurrent requests)
- Consider using socat or nc for simplicity (proven tools)
- JSON protocol for easy debugging (human-readable messages)

---

#### Risk 4: X11 Dependency
**Likelihood**: High | **Impact**: Medium

**Description**: Testing requires X11 environment, complicating automation.

**Mitigation**:
- Use Xvfb for headless testing (virtual X server)
- Mock bspc commands in unit tests (test logic without bspwm)
- Separate testable logic from X11 interaction (pure functions)
- Document manual testing procedures (for integration tests)
- Consider containerized test environment (Docker with Xvfb + bspwm)

---

#### Risk 5: State Synchronization
**Likelihood**: Medium | **Impact**: Medium

**Description**: Daemon state could desync from actual bspwm state.

**Mitigation**:
- Minimize stateful caching (query bspwm as source of truth)
- Implement cache invalidation on events (clear cache on node_add, etc.)
- Periodic state reconciliation (every 60 seconds, full resync)
- Query bspwm as source of truth (never trust cache alone)
- Add state validation checks (detect desyncs, log warnings)

---

#### Risk 6: Performance Degradation
**Likelihood**: Low | **Impact**: Medium

**Description**: Single daemon handling all events might be slower than separate daemons.

**Mitigation**:
- Event handling should be minimal (quick handlers, no blocking)
- Use caching to reduce bspc calls (5 second TTL for queries)
- Profile and benchmark (measure event handling latency)
- Implement event batching (group rapid events, process once)
- Optimize hot paths (balance, desktops are most frequent)

---

### Compatibility Risks

#### Risk 1: Breaking User Workflows
**Likelihood**: High | **Impact**: High

**Description**: Changing from separate utilities to unified CLI breaks scripts and keybindings.

**Mitigation**:
- Maintain backward-compatible shims (bsp-balance wrapper → bsp balance on)
- Document migration path clearly (step-by-step guide)
- Provide migration script (auto-update keybindings in sxhkdrc)
- Keep old binaries as deprecated wrappers (transition period)
- Announce breaking changes clearly (CHANGELOG, README)

---

#### Risk 2: Configuration Migration
**Likelihood**: High | **Impact**: Medium

**Description**: Moving from text files to JSON config loses user settings.

**Mitigation**:
- Auto-migration script (convert text files to JSON on first run)
- Preserve old config format as fallback (read old files if JSON missing)
- Import old configs to new format (bsp config migrate command)
- Document config changes (side-by-side comparison)
- Provide config examples (default config with comments)

---

#### Risk 3: Systemd Service Changes
**Likelihood**: High | **Impact**: Medium

**Description**: Replacing 5 services with 1 requires user intervention.

**Mitigation**:
- Provide systemd migration script (disable old, enable new)
- Auto-disable old services (migration script does this)
- Auto-enable new service (migration script does this)
- Document service changes (clear instructions)
- Test on fresh install (verify no conflicts)

---

#### Risk 4: Library Dependencies
**Likelihood**: Medium | **Impact**: High

**Description**: Requiring new libraries (_events, _lifecycle, _config, _bspwm) that users might not have.

**Mitigation**:
- Bundle required libraries with package (include in .local/libexec/bsp/lib/)
- Check for library availability on start (graceful failure if missing)
- Graceful degradation if library missing (disable features, not crash)
- Document dependencies clearly (README, installation guide)
- Provide installation script (install all dependencies)

---

### User Adoption Risks

#### Risk 1: Learning Curve
**Likelihood**: High | **Impact**: Medium

**Description**: New CLI interface requires learning new commands.

**Mitigation**:
- Comprehensive documentation (man pages, README, examples)
- Quick start guide (get running in 5 minutes)
- Example workflows (common tasks, copy-paste commands)
- Tab completion (zsh and bash, discover commands easily)
- Helpful error messages (suggest correct command)
- Interactive tutorial (bsp help-interactive, future feature)

---

#### Risk 2: Feature Discovery
**Likelihood**: Medium | **Impact**: Low

**Description**: Users might not discover new features (layout management, advanced flag handlers).

**Mitigation**:
- Clear feature documentation (README, man pages)
- `bsp help` lists all features and commands
- Release notes highlighting new features (CHANGELOG)
- Example configurations (showcase features)
- Community sharing (dotfiles examples, blog posts)

---

#### Risk 3: Migration Friction
**Likelihood**: Medium | **Impact**: Medium

**Description**: Users might resist migrating from working setup.

**Mitigation**:
- Emphasize benefits (performance, features, unified CLI)
- Make migration easy (scripts, docs, shims)
- Maintain backward compatibility (old command names work)
- Provide support channel (GitHub issues, discussions)
- Phase deprecation slowly (6-month transition period)

---

#### Risk 4: Debugging Difficulty
**Likelihood**: Medium | **Impact**: Medium

**Description**: Unified daemon harder to debug than separate utilities.

**Mitigation**:
- Comprehensive logging (debug mode, event logging)
- `bsp status` shows detailed state (feature status, health)
- `bsp daemon status` shows daemon health (uptime, crashes)
- Debug mode with verbose output (--debug flag)
- Event logging for troubleshooting (replay events)
- Clear error messages (actionable, not cryptic)

---

### Operational Risks

#### Risk 1: Single Point of Failure
**Likelihood**: Low | **Impact**: High

**Description**: Single daemon crash disables all features.

**Mitigation**:
- Robust error handling (catch all exceptions)
- Feature isolation (one feature crash doesn't kill daemon)
- Systemd auto-restart (Restart=always in service file)
- Health monitoring (detect hung features, restart them)
- Graceful degradation (disable failed feature, continue others)

---

#### Risk 2: Resource Usage
**Likelihood**: Low | **Impact**: Low

**Description**: Single daemon might use more resources than needed if only one feature used.

**Mitigation**:
- Feature enable/disable at runtime (only subscribe to needed events)
- Only subscribe to needed events (if balance disabled, don't subscribe to node_add)
- Efficient event handling (fast handlers, minimal work)
- Memory profiling (ensure no leaks)
- Lazy loading (load features on demand)

---

#### Risk 3: Upgrade Path
**Likelihood**: Medium | **Impact**: Medium

**Description**: Future upgrades might break compatibility.

**Mitigation**:
- Semantic versioning (major.minor.patch)
- Config version tracking (detect old config, auto-migrate)
- Auto-migration on upgrade (run migration script)
- Deprecation warnings (announce breaking changes early)
- Changelog documentation (clear upgrade notes)

---

## Success Criteria

### Functional Requirements
- [ ] All 8 existing utilities work with identical functionality
- [ ] Single daemon replaces 5 separate daemons
- [ ] Unified CLI interface (`bsp <subcommand>`)
- [ ] Configuration via JSON file
- [ ] Features can be enabled/disabled at runtime
- [ ] Status reporting (`bsp status`)
- [ ] Tree visualization (`bsp tree`)
- [ ] Layout management (save/load/apply)
- [ ] Rofi integration for interactive menus
- [ ] All 6 flag types implemented (marked, floating, hidden, sticky, locked, private)

### Performance Requirements
- [ ] Event handling latency < 50ms (from event to handler completion)
- [ ] Memory usage < 50MB for daemon (with all features enabled)
- [ ] CPU usage < 1% idle, < 5% during events
- [ ] Startup time < 1 second (daemon start to ready)
- [ ] No noticeable performance degradation vs old utilities

### Quality Requirements
- [ ] Zero crashes during normal operation (daemon stays running)
- [ ] Graceful handling of bspwm not running (helpful error, not crash)
- [ ] Comprehensive error messages (helpful, actionable)
- [ ] All features have tests (unit + integration)
- [ ] Code coverage > 70% (excluding external command wrappers)
- [ ] No known bugs (all reported bugs fixed)

### Documentation Requirements
- [ ] README with installation and quick start
- [ ] Man pages for all commands
- [ ] Configuration guide with examples
- [ ] Architecture documentation (this document)
- [ ] Migration guide from old version
- [ ] Example workflows (common tasks)
- [ ] Troubleshooting guide (common issues)

### User Experience Requirements
- [ ] `--help` for all commands (helpful output)
- [ ] Tab completion (zsh and bash)
- [ ] Backward compatibility shims for old binaries
- [ ] Clear migration path (documented, scripted)
- [ ] Helpful error messages with suggestions
- [ ] Consistent command naming and structure
- [ ] Beautiful output (colors, formatting, Unicode)

### Operational Requirements
- [ ] Systemd service file (bsp.service)
- [ ] Auto-start on login (optional, user choice)
- [ ] Log rotation (prevent log files from growing indefinitely)
- [ ] Health monitoring (detect and recover from failures)
- [ ] Auto-recovery from crashes (systemd Restart=always)
- [ ] Config validation (catch errors early)

---

## Conclusion

This design specification provides a comprehensive blueprint for rewriting the bsp package from 8 standalone utilities into a unified, modern window management system.

### Key Achievements
- **Architecture**: Unified daemon + CLI dispatcher pattern
- **Reduced Overhead**: 5 daemons → 1 daemon (80% reduction)
- **Unified Interface**: 8 separate commands → 1 CLI with subcommands
- **Enhanced Features**: All existing features + layout management + complete flag handlers
- **Better UX**: Status reporting, tree visualization, configuration management
- **Library-first**: Reusable _bspwm and _layout libraries

### Next Steps
1. **Review and approve this design** (stakeholder sign-off)
2. **Set up development environment** (bspwm test instance, Xvfb)
3. **Begin Phase 1** (create _bspwm library)
4. **Iterate through phases** (weekly releases)
5. **Beta testing** (internal dogfooding)
6. **Release v2.0.0** (production-ready)

### Expected Outcomes
- **Short-term** (1 month): Unified, polished bsp utility with all features working better
- **Medium-term** (3 months): Advanced features in use, community contributions easier
- **Long-term** (6+ months): bsp becomes reference implementation for dotfiles utilities

**Recommendation**: **Proceed with implementation, following phased roadmap.**

---

*Design completed: 2025-11-07*
*Estimated effort: 4-5 weeks*
*Architect: Claude (Sonnet 4.5)*
