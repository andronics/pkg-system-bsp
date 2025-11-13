# BSP - Design Summary

## Overview

**Package**: bsp (Binary Space Partitioning Window Manager Utilities)

**Current State**: 8 standalone utilities (5 daemons, 3 CLI tools)

**Target State**: Unified daemon + CLI dispatcher with git-like interface

**Timeline**: 4-5 weeks

**Effort**: Moderate complexity

---

## Architecture

### Design Pattern
**Unified Daemon + CLI Dispatcher**

```
User → bsp CLI → IPC → bspd daemon → Event bus → Features
                  ↓                      ↓
            Direct commands         bspwm (via _bspwm)
```

### Core Components

1. **Main Dispatcher** (`bsp`): Git-like CLI with subcommands
2. **Daemon** (`bspd`): Single event loop, IPC server, feature manager
3. **Features**: Modular plugins (balance, desktops, flag, floating-borders, ventilate)
4. **Libraries**: _bspwm (NEW), _layout (NEW), _events, _lifecycle, _config, _cache

### File Structure

```
bsp/
├── .local/bin/bsp                    # Main CLI dispatcher
└── .local/libexec/bsp/
    ├── bspd                          # Daemon
    ├── bsp-daemon                    # Daemon control
    ├── bsp-status                    # Status reporting
    ├── bsp-feature                   # Feature control
    ├── bsp-side                      # Directional placement
    ├── bsp-shroud                    # Hide/show windows
    ├── bsp-fullscreen                # Fullscreen toggle
    ├── bsp-layout                    # Layout management
    ├── bsp-tree                      # Tree visualization
    ├── bsp-config                    # Config management
    └── features/
        ├── balance.zsh               # Auto-balance
        ├── desktops.zsh              # Dynamic desktops
        ├── flag.zsh                  # Flag reactions
        ├── floating-borders.zsh      # Border removal
        └── ventilate.zsh             # Terminal swallowing
```

---

## Command Structure

### Git-like CLI

```bash
bsp daemon start|stop|restart|status   # Daemon management
bsp status [--json] [--watch]          # Comprehensive status

bsp balance on|off|status              # Feature control
bsp desktops on|off|status|config
bsp flag on|off|status|config
bsp floating-borders on|off|status
bsp ventilate on|off|status|config

bsp side <direction>                   # Direct commands
bsp shroud cover|reveal
bsp fullscreen

bsp layout list|save|load|apply        # Layout management
bsp tree [options]                     # Tree visualization
bsp config get|set|edit|validate       # Config management
```

### Example Usage

```bash
# Start daemon
bsp daemon start

# Enable features
bsp balance on
bsp ventilate on

# Check status
bsp status

# Place window directionally
bsp side west

# Save and apply layout
bsp layout save my-layout
bsp layout apply my-layout

# Interactive layout selector
bsp layout select

# Visualize window tree
bsp tree --focused
```

---

## Key Features

### Current Features (Preserved)
- **balance**: Auto-balance window tree on events
- **desktops**: Dynamic desktop management (auto-add/remove)
- **flag**: React to window flag changes (marked, floating, etc.)
- **floating-borders**: Remove borders from floating windows
- **ventilate**: Terminal window swallowing
- **side**: Directional window placement
- **shroud**: Hide/show windows and polybar
- **fullscreen**: Toggle fullscreen state

### New Features (Added)
- **Unified CLI**: Git-like interface with `bsp <subcommand>`
- **IPC**: CLI communicates with daemon via Unix socket
- **Status**: Comprehensive status reporting (`bsp status`)
- **Tree**: ASCII art window tree visualization (`bsp tree`)
- **Layout Management**: Save/load/apply window layouts
- **Config Management**: JSON configuration with validation
- **Complete Flag Handlers**: All 6 flag types implemented (not just marked)
- **Rofi Integration**: Interactive menus for layouts and features

### Enhanced Features
- **Smart Balance**: Configurable strategies, exclude classes
- **Desktop Auto-naming**: Semantic desktop names
- **Better Ventilate**: Improved class detection
- **Health Monitoring**: Auto-restart failed features
- **Event Logging**: Debug and troubleshoot easily

---

## Library Integration

### Core Libraries (Required)

#### _bspwm (NEW - High Priority)
Complete bspwm/bspc abstraction layer

**Functions**:
- Query: nodes, desktops, monitors, focused, tree
- Manipulation: set state/flag/border/color, balance, preselect
- Desktop: add, remove, rename, focus, swap
- Window properties: class, name, command, PID (via xprop)
- Utility: is-running, version, validate-selector

**Why**: Centralize bspwm interaction, error handling, caching, testing

#### _lifecycle
Feature lifecycle management (start, stop, restart, health)

#### _events
Event bus for routing bspwm events to features

#### _config
JSON configuration loading, validation, runtime updates

#### _cache
Cache query results for performance

#### _log, _singleton, _args, _xdg
Already in use, continue using

### Optional Libraries (Enhanced Features)

#### _layout (NEW - Medium Priority)
Layout profile management

**Functions**:
- list, save, load, apply, delete, export, import, validate

**Why**: Save/restore window layouts, reproducible workspaces

#### _rofi
Interactive Rofi menus (layout selector, feature toggler)

#### _ui
Terminal UI formatting (tables, trees, status indicators, colors)

#### _notify
Desktop notifications (desktop events, layout changes, errors)

---

## Configuration

### JSON Configuration Schema

**File**: `~/.config/bsp/config.json`

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
      "start_id": 1,
      "auto_naming": false
    },
    "flag": {
      "enabled": true,
      "marked": {
        "border_width": 3,
        "border_color": "#0000ff"
      },
      "floating": {...},
      "hidden": {...},
      "sticky": {...},
      "locked": {...},
      "private": {...}
    },
    "floating_borders": {
      "enabled": true,
      "border_width": 0
    },
    "ventilate": {
      "enabled": true,
      "terminals": ["Alacritty", "terminal-drop", "Code"],
      "no_inhale": ["Rofi", "xev", "firefox"]
    }
  },
  "ipc": {
    "socket": "${XDG_RUNTIME_DIR}/bsp.sock",
    "timeout": 5
  },
  "logging": {
    "level": "INFO",
    "file": "${XDG_STATE_HOME}/bsp/bsp.log"
  },
  "layouts": {
    "directory": "${XDG_CONFIG_HOME}/bsp/layouts"
  }
}
```

### Migration
- Auto-migrate from text files (no_inhale, terminals) to JSON
- Preserve old config as fallback
- Migration script: `bsp config migrate`

---

## Implementation Phases

### Phase 1: Foundation (Week 1)
- Create _bspwm library
- Create main dispatcher (bsp)
- Create daemon foundation (bspd)
- Migrate one feature (balance)

**Deliverable**: Working daemon with balance feature

### Phase 2: Feature Migration (Week 2)
- Migrate all 5 daemon features (desktops, flag, floating-borders, ventilate)
- Migrate 3 CLI utilities (side, shroud, fullscreen)
- Implement JSON configuration system
- Config migration script

**Deliverable**: All features working, single daemon, JSON config

### Phase 3: Enhancement (Week 3)
- Complete IPC system
- Status reporting (bsp status)
- Tree visualization (bsp tree)
- Improve features (complete flag handlers, smart balance)
- Error handling and graceful degradation

**Deliverable**: Enhanced features, IPC working, status and tree commands

### Phase 4: Advanced Features (Week 4)
- Create _layout library
- Layout management commands
- Rofi integration
- Comprehensive documentation
- Testing and polish

**Deliverable**: Layout management, Rofi menus, complete documentation

### Phase 5: Polish & Release (Bonus Week)
- Systemd integration
- Migration script and guide
- Performance optimization
- UI polish
- Release preparation

**Deliverable**: Production-ready v2.0.0 release

---

## Benefits

### For Users
- **Unified Interface**: One CLI, not 8 separate commands
- **Discoverability**: `bsp help` shows all features
- **Status Visibility**: `bsp status` shows what's running
- **Configuration**: JSON config, not script editing
- **New Features**: Layout management, tree visualization
- **Better Errors**: Helpful, actionable error messages

### For Developers
- **Maintainability**: Shared code, not duplication
- **Testability**: Mock bspwm, test features in isolation
- **Extensibility**: Easy to add new features
- **Reusability**: _bspwm library benefits all bspwm utilities
- **Debugging**: Event logging, status reporting

### For Operations
- **Resource Efficiency**: 5 daemons → 1 daemon (80% reduction)
- **Reliability**: Health monitoring, auto-restart
- **Observability**: Comprehensive logging, status
- **Management**: One systemd service, not 5

---

## Risks and Mitigations

### Technical Risks
- **Event handling complexity**: Mitigated by event bus pattern, testing
- **bspwm dependency**: Mitigated by _bspwm abstraction, error handling
- **IPC complexity**: Mitigated by simple Unix sockets, timeouts

### Compatibility Risks
- **Breaking workflows**: Mitigated by backward-compatible shims, migration script
- **Config migration**: Mitigated by auto-migration, fallback to old format
- **Systemd changes**: Mitigated by migration script

### User Adoption Risks
- **Learning curve**: Mitigated by documentation, tab completion, examples
- **Migration friction**: Mitigated by easy migration, backward compatibility

---

## Success Criteria

### Functional
- [ ] All 8 utilities work identically
- [ ] Single daemon replaces 5 daemons
- [ ] Unified CLI interface
- [ ] JSON configuration
- [ ] Runtime feature toggling
- [ ] Layout management
- [ ] Complete flag handlers

### Performance
- [ ] Event latency < 50ms
- [ ] Memory < 50MB
- [ ] CPU < 1% idle, < 5% during events
- [ ] Startup < 1 second

### Quality
- [ ] Zero crashes
- [ ] Graceful error handling
- [ ] Comprehensive error messages
- [ ] Code coverage > 70%

### Documentation
- [ ] README with quick start
- [ ] Man pages for all commands
- [ ] Configuration guide
- [ ] Migration guide

---

## Conclusion

The bsp rewrite transforms a collection of useful but fragmented utilities into a cohesive, powerful window management system.

**Key Improvements**:
- 80% reduction in daemon processes (5 → 1)
- Unified CLI interface (8 commands → 1 CLI with subcommands)
- Advanced features (layout management, tree visualization)
- Better maintainability (shared code, libraries)
- Enhanced user experience (status, config, help)

**Timeline**: 4-5 weeks

**Recommendation**: Proceed with implementation

---

*Design Summary*
*Document version: 1.0*
*Date: 2025-11-07*
