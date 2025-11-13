# BSP - Complete Rewrite Analysis

## Executive Summary

**Current State**: The bsp package is a collection of 8 standalone bspwm enhancement utilities that provide advanced window management behaviors. While functional, the package lacks cohesion, has significant technical debt, and misses opportunities for modern architecture patterns.

**Rewrite Recommendation**: **YES - PROCEED**

**Estimated Effort**: **3-4 weeks (Moderate Complexity)**

**Key Benefits**:
- Unified CLI interface with git-like dispatcher pattern
- Shared state management and event handling infrastructure
- Comprehensive configuration system
- Advanced features: layout profiles, workspace automation, monitor management
- Better error handling and logging
- Plugin-style architecture for extensibility
- Reduced code duplication

**Major Risks**:
- Breaking backward compatibility for systemd services
- User workflow disruption (current users expect standalone daemons)
- Complex event handling coordination
- Testing complexity with X11/bspwm environment

---

## Current Implementation Analysis

### Functionality Overview

The bsp package provides 8 distinct utilities that enhance bspwm window management:

#### 1. **bsp-balance** (Event-driven daemon)
- **Purpose**: Automatically balances window tree nodes
- **How**: Subscribes to bspwm events (node_add, node_remove, node_state, node_geometry)
- **Logic**: For each event, balances all windows on desktop and non-window descendant nodes
- **Use Case**: Maintains equal window sizes automatically

#### 2. **bsp-desktops** (Event-driven daemon)
- **Purpose**: Dynamic desktop management (auto-add/remove)
- **How**: Maintains exactly 1 empty desktop at all times
- **Logic**:
  - Adds new desktop when all are occupied
  - Removes unoccupied desktops when more than 1 empty exists
  - Starts desktop ID counter at 1
- **Use Case**: Infinite desktop scrolling without manual creation

#### 3. **bsp-flag** (Event-driven daemon)
- **Purpose**: React to window flag changes
- **How**: Subscribes to node_flag events
- **Current Implementation**: Only "marked" flag is implemented
  - Changes border width to 3 and color to blue when marked
  - Resets border width to 1 and color to red when unmarked
- **Unimplemented**: floating, hidden, locked, private, sticky flags
- **Dependencies**: Uses `palette` command for colors, `common-unimplemented` for stubs

#### 4. **bsp-floating-borders** (Event-driven daemon)
- **Purpose**: Remove borders from floating windows
- **How**: Subscribes to node_state events
- **Logic**: When window enters floating state, sets border_width to 0
- **Use Case**: Clean aesthetic for floating windows

#### 5. **bsp-side** (CLI utility, not a daemon)
- **Purpose**: Directional window placement with smart tiling
- **How**: Takes direction argument (west/north/east/south)
- **Logic**:
  - Focuses appropriate split (first/second child)
  - Ensures correct split orientation (vertical/horizontal)
  - Enforces tiled state
- **Use Case**: Vim-like directional window placement
- **XDG Support**: Uses `_xdg` library for configuration directories

#### 6. **bsp-shroud** (CLI utility, not a daemon)
- **Purpose**: Hide/show windows and polybar simultaneously
- **Commands**:
  - `cover`: Hides polybar and all leaf nodes on focused desktop
  - `reveal`: Shows polybar and all leaf nodes on focused desktop
- **Use Case**: Presentation mode, distraction-free mode
- **XDG Support**: Uses `_xdg` library for configuration directories

#### 7. **bsp-ventilate** (Event-driven daemon)
- **Purpose**: Terminal window swallowing (hide terminal when launching GUI)
- **How**: Subscribes to node_add and node_remove events
- **Logic**:
  - "Inhale": When GUI app launches from terminal, hide the terminal
  - "Exhale": When GUI app closes, restore the terminal
  - Uses WM_CLASS and WM_COMMAND to identify windows
- **Configuration**:
  - `no_inhale`: Classes that should never inhale (Rofi, xev, xprop, gcr-prompter, firefox, Google-chrome, Code)
  - `terminals`: Classes considered terminals (terminal-drop, Code, Alacritty)
- **State**: Maintains `inhaled` state file tracking hidden terminals
- **Inspiration**: Based on @JopStrow/bspswallow
- **XDG Support**: Uses `_xdg` library for configuration directories

#### 8. **bsp-fullscreen** (CLI utility, not a daemon)
- **Purpose**: Toggle fullscreen state
- **Logic**: Simple toggle between tiled and fullscreen
- **Use Case**: Keybinding-friendly fullscreen toggle

### Architecture Analysis

**Current Design Pattern**: Collection of independent standalone utilities

**Characteristics**:
- **No central dispatcher**: Each utility is completely independent
- **No shared state**: Each daemon maintains its own state
- **No coordination**: Daemons don't communicate with each other
- **Inconsistent patterns**:
  - 5 event-driven daemons (balance, desktops, flag, floating-borders, ventilate)
  - 3 CLI utilities (side, shroud, fullscreen)
- **Library usage varies**:
  - All use `_log`
  - Daemons use `_singleton` and `_onexit`
  - Some use `_xdg` (side, shroud, ventilate)
  - One uses `_string` (ventilate)
  - One uses `_args` (side, shroud, ventilate)
  - One uses `_common` (flag)

**File Structure**:
```
bsp/
├── .local/
│   ├── bin/
│   │   ├── bsp-balance              # Daemon (event subscriber)
│   │   ├── bsp-desktops             # Daemon (event subscriber)
│   │   ├── bsp-flag                 # Daemon (event subscriber)
│   │   ├── bsp-floating-borders     # Daemon (event subscriber)
│   │   ├── bsp-ventilate            # Daemon (event subscriber)
│   │   ├── bsp-side                 # CLI utility
│   │   ├── bsp-shroud               # CLI utility (subcommands)
│   │   └── bsp-fullscreen           # CLI utility
│   └── share/
│       └── systemd/
│           └── user/
│               ├── bsp-balance.service
│               ├── bsp-desktops.service
│               ├── bsp-flag.service
│               ├── bsp-floating-borders.service
│               └── bsp-ventilate.service
└── .config/
    └── bsp-ventilate/
        ├── no_inhale                 # Class blacklist
        └── terminals                 # Terminal class list
```

**Code Organization**:
- Monolithic scripts (no modularization)
- Each utility is self-contained
- No code reuse between utilities
- No shared event handling infrastructure

**Complexity Assessment**:
- **Individual utilities**: Low-Medium complexity
- **Overall package**: Medium complexity
- **Integration potential**: High (many opportunities for shared infrastructure)

### Dependencies

#### External Commands
- **bspc** (bspwm client): Core dependency for all window management
  - Used for: queries, subscriptions, node manipulation, desktop management, configuration
  - Commands used: query, subscribe, node, desktop, monitor, config
- **xprop**: Window property inspection (used by bsp-flag, bsp-ventilate)
- **polybar-msg**: Polybar control (used by bsp-shroud)
- **grep**: Pattern matching (used by bsp-ventilate)
- **wc**: Line counting (used by bsp-desktops)
- **awk**: Text processing (used by bsp-flag, bsp-ventilate)
- **sed**: Text processing (used by bsp-ventilate)

#### Library Usage (Current)
- **_log**: Logging (all utilities)
- **_singleton**: Single-instance enforcement (5 daemons)
- **_onexit**: Cleanup handlers (5 daemons)
- **_xdg**: XDG directory setup (side, shroud, ventilate)
- **_args**: Argument parsing (side, shroud, ventilate)
- **_string**: String manipulation (ventilate)
- **_common**: Common utilities (flag)

#### External Dependencies (Not in Use)
- **palette**: Color scheme access (referenced in bsp-flag)

#### System Requirements
- bspwm window manager (active and running)
- X11 display server
- systemd (for daemon management)
- Zsh shell

#### Configuration Files
- `~/.config/bsp-ventilate/no_inhale`: Class exclusion list
- `~/.config/bsp-ventilate/terminals`: Terminal class list
- `~/.local/share/bsp-ventilate/inhaled`: State file for swallowed windows

### Quality Assessment

**Code Quality Score**: **6/10**

**Strengths**:
- Clean, readable code
- Consistent naming conventions
- Good use of logging
- Proper singleton pattern for daemons
- XDG directory support in newer utilities

**Weaknesses**:
- No error handling (commands can fail silently)
- No input validation
- Hardcoded values (border widths, colors)
- Incomplete implementations (bsp-flag has 5 unimplemented handlers)
- No command-line help/usage
- Inconsistent library usage patterns
- Magic numbers and strings

**Documentation Quality**: **2/10**
- No README
- No man pages
- No inline documentation
- No usage examples
- Only source code comments are attribution (bsp-ventilate)

**Test Coverage**: **0/10**
- No tests
- No test infrastructure
- Difficult to test (requires running bspwm environment)

**Performance Issues**:
- Event subscribers run continuously (CPU overhead)
- No event batching/debouncing
- Potentially redundant work (balance rebalances entire tree on every event)
- State file I/O on every ventilate operation

**Technical Debt**:
1. **Incomplete implementations**: bsp-flag has 5 unimplemented flag handlers
2. **No configuration system**: Hardcoded values scattered throughout
3. **No unified logging**: Each daemon logs independently
4. **No shared event infrastructure**: Each daemon subscribes separately
5. **State management**: Primitive (text files for ventilate)
6. **No dependency management**: Assumes all commands available
7. **No versioning**: No version tracking
8. **No migration path**: Configuration format changes would break things

### Pain Points

#### User-Facing Pain Points
1. **No discoverability**: No way to list available utilities or see what's running
2. **No status visibility**: Can't check if daemons are working
3. **No configuration**: Can't customize behavior without editing scripts
4. **Inconsistent interfaces**: Some are daemons, some are CLI, no unified approach
5. **No help system**: No `--help` flags or documentation
6. **Service management complexity**: Must manually enable/disable systemd services
7. **Debugging difficulty**: When things break, hard to diagnose

#### Developer Pain Points
1. **Code duplication**: Event subscription boilerplate repeated 5 times
2. **No testing strategy**: Changes require manual testing in live environment
3. **No shared utilities**: Common bspwm operations reimplemented
4. **Inconsistent patterns**: Hard to maintain coherent style
5. **No modularity**: Can't reuse components
6. **State management primitive**: Text file parsing is brittle

#### Operational Pain Points
1. **Resource usage**: 5 separate daemon processes
2. **No coordination**: Daemons don't know about each other
3. **Race conditions**: Multiple daemons reacting to same events
4. **No graceful degradation**: If bspwm isn't running, daemons hang/crash
5. **Log fragmentation**: Logs scattered across multiple processes

---

## Backend Technology Assessment

### Current Backend: bspwm + bspc

**What It Is**:
- **bspwm**: Binary Space Partitioning Window Manager for X11
- **bspc**: Command-line client for controlling bspwm
- Written in C, highly efficient
- Minimalist philosophy (no built-in bar, menus, etc.)

**How It Works**:
- **Tree Structure**: Windows organized as leaves in binary tree
- **Event System**: bspc subscribe provides real-time events
- **Socket-Based IPC**: bspc communicates via Unix socket
- **Selector Syntax**: Powerful query language for targeting nodes/desktops/monitors

**Events Available** (via `bspc subscribe`):
- `node_add`: New window created
- `node_remove`: Window destroyed
- `node_state`: Window state changed (tiled, floating, fullscreen, etc.)
- `node_flag`: Window flag changed (marked, locked, sticky, private, hidden)
- `node_geometry`: Window size/position changed
- `node_focus`: Focus changed
- `node_activate`: Window activated
- `desktop_add`: Desktop created
- `desktop_remove`: Desktop removed
- `desktop_focus`: Desktop focus changed
- `monitor_add`: Monitor connected
- `monitor_remove`: Monitor disconnected
- `monitor_focus`: Monitor focus changed

**Capabilities**:
- Window manipulation (move, resize, swap, rotate, flip)
- Desktop management (add, remove, rename, swap)
- Monitor management (add, remove, focus, rename)
- Node querying (powerful selector syntax)
- Configuration (runtime and persistent)
- Preselection (manual tiling control)
- Receptacles (placeholder nodes)

**Limitations**:
- X11 only (no Wayland support)
- No built-in layouts (users must script them)
- No built-in window rules (must be scripted)
- No session management
- Minimal built-in automation

**Performance**:
- Extremely lightweight
- Fast event delivery
- Low memory footprint
- Efficient tree operations

### Alternative Backends

#### Option 1: i3/sway
- **Pros**: Better built-in features, Wayland support (sway), IPC protocol
- **Cons**: Different paradigm (containers vs tree), would require complete rewrite
- **Verdict**: Not suitable (fundamentally different architecture)

#### Option 2: Hyprland
- **Pros**: Modern Wayland compositor, dynamic tiling, animations
- **Cons**: Different API, immature ecosystem, no X11 support
- **Verdict**: Not suitable (incompatible with current user base)

#### Option 3: xdotool/wmctrl
- **Pros**: Generic X11 window manipulation
- **Cons**: No event system, limited window manager integration, slow
- **Verdict**: Not suitable (missing core event functionality)

### Recommended Backend for Rewrite

**RECOMMENDATION**: **Stick with bspwm/bspc**

**Justification**:
1. **Event System**: bspc subscribe is perfectly suited for reactive behaviors
2. **Powerful Queries**: Selector syntax provides all needed functionality
3. **Performance**: Extremely lightweight and fast
4. **Maturity**: Stable, well-maintained, unlikely to break
5. **User Base**: Users chose bspwm for minimalism, don't change paradigm
6. **Compatibility**: Existing bspwm configs work with enhanced utilities

**Trade-offs**:
- **X11 Only**: Can't support Wayland (acceptable for target audience)
- **Manual Scripting**: Need to build abstractions (opportunity for library!)
- **Testing Complexity**: Requires X11 environment (mitigated with Xvfb)

### System Integration

#### Systemd Services
- **Current**: 5 independent user services
- **Opportunity**: Single service with multiple workers, or systemd templates

#### D-Bus Interfaces
- **Current**: None
- **Opportunity**: D-Bus interface for IPC between utilities and external tools

#### Configuration Files
- **Current**: Minimal (only bsp-ventilate has config files)
- **Opportunity**: Comprehensive JSON/TOML configuration system

#### Protocols
- **bspc Protocol**: Unix socket communication
- **X11 Protocol**: Window property inspection via xprop

---

## Library Ecosystem Integration

### Relevant Libraries (Existing)

The following library extensions are highly relevant for the bsp rewrite:

#### Core Infrastructure

**_log** (Already in use)
- **Current Use**: Basic logging in all utilities
- **Enhancement Opportunity**: Structured logging with levels, contexts, log rotation
- **Integration**: Central logging for unified daemon

**_singleton** (Already in use)
- **Current Use**: Prevent multiple daemon instances
- **Enhancement Opportunity**: PID tracking, graceful handoff
- **Integration**: Shared singleton for unified daemon

**_onexit** (Already in use)
- **Current Use**: Cleanup handlers for daemons
- **Enhancement Opportunity**: Coordinated shutdown
- **Integration**: Central cleanup orchestration

**_xdg** (Already in use)
- **Current Use**: XDG directory setup (3 utilities)
- **Enhancement Opportunity**: Consistent paths across all utilities
- **Integration**: Package-wide XDG structure
- **Functions Needed**:
  - `xdg-setup-dirs`: Already used
  - `xdg-cache-path`, `xdg-config-path`, `xdg-state-path`: For path construction

**_args** (Already in use)
- **Current Use**: Argument parsing (3 utilities)
- **Enhancement Opportunity**: Standardized CLI interface, subcommand parsing
- **Integration**: Main dispatcher argument handling
- **Functions Needed**:
  - Flag parsing (`--help`, `--version`, `--debug`)
  - Subcommand routing
  - Positional argument handling

**_string** (Already in use)
- **Current Use**: String manipulation (bsp-ventilate)
- **Enhancement Opportunity**: More string utilities
- **Integration**: Window class matching, string formatting
- **Functions Needed**:
  - `string-lowercase`: Already used
  - `string-match`, `string-contains`: For class matching

**_common** (Already in use)
- **Current Use**: Common utilities (bsp-flag)
- **Enhancement Opportunity**: Shared bspwm abstractions
- **Integration**: bspwm operation helpers
- **Functions Needed**:
  - `common-unimplemented`: Already used (but should be removed!)

#### Event & State Management

**_events** (NOT currently in use - HIGH PRIORITY)
- **Why It's Needed**: Centralized event bus for bspwm events
- **How It Would Be Used**:
  - Single bspc subscribe process
  - Event routing to multiple handlers
  - Event filtering and transformation
  - Event debouncing/throttling
- **Functions Needed**:
  - `event-subscribe <event-type> <handler>`: Register event handler
  - `event-emit <event-type> <data>`: Emit event (for testing)
  - `event-unsubscribe <handler-id>`: Remove handler
  - `event-filter <pattern>`: Filter events
  - `event-debounce <interval>`: Rate limiting
- **Integration Benefits**:
  - Single event loop for all features
  - Reduced resource usage
  - Easier testing (mock events)
  - Event logging/replay

**_lifecycle** (NOT currently in use - HIGH PRIORITY)
- **Why It's Needed**: Daemon lifecycle management
- **How It Would Be Used**:
  - Start/stop/restart features
  - Health checking
  - Automatic recovery
  - Feature enable/disable
- **Functions Needed**:
  - `lifecycle-start <feature>`: Start feature
  - `lifecycle-stop <feature>`: Stop feature
  - `lifecycle-restart <feature>`: Restart feature
  - `lifecycle-status <feature>`: Check status
  - `lifecycle-health`: Health check
- **Integration Benefits**:
  - Runtime feature toggling
  - Graceful degradation
  - Better debugging

**_cache** (NOT currently in use - MEDIUM PRIORITY)
- **Why It's Needed**: Performance optimization
- **How It Would Be Used**:
  - Cache bspc query results
  - Cache window properties
  - Cache computed layouts
  - TTL-based invalidation
- **Functions Needed**:
  - `cache-get <key>`: Retrieve value
  - `cache-set <key> <value> <ttl>`: Store value
  - `cache-invalidate <pattern>`: Clear cache
  - `cache-clear`: Clear all
- **Integration Benefits**:
  - Reduce bspc calls
  - Faster event handling
  - Lower CPU usage

#### Configuration Management

**_config** (NOT currently in use - HIGH PRIORITY)
- **Why It's Needed**: Unified configuration system
- **How It Would Be Used**:
  - JSON/TOML config file parsing
  - Default values
  - Config validation
  - Runtime config updates
  - Profile support
- **Functions Needed**:
  - `config-load <file>`: Load configuration
  - `config-get <key> [default]`: Get value
  - `config-set <key> <value>`: Set value
  - `config-validate <schema>`: Validate config
  - `config-merge <file>`: Merge configs
- **Configuration Structure**:
  ```json
  {
    "features": {
      "balance": { "enabled": true },
      "desktops": { "enabled": true, "start_id": 1 },
      "flag": {
        "enabled": true,
        "marked": { "border_width": 3, "border_color": "blue" }
      },
      "floating_borders": { "enabled": true, "border_width": 0 },
      "ventilate": {
        "enabled": true,
        "no_inhale": ["Rofi", "xev", "firefox"],
        "terminals": ["Alacritty", "terminal-drop"]
      }
    },
    "layouts": {
      "default": { "split_ratio": 0.5 },
      "dev": { "split_ratio": 0.618 }
    }
  }
  ```
- **Integration Benefits**:
  - User-friendly configuration
  - No script editing
  - Profile switching
  - Config validation

#### UI Integration

**_rofi** (NOT currently in use - MEDIUM PRIORITY)
- **Why It's Needed**: Interactive UI for complex operations
- **How It Would Be Used**:
  - Layout selection
  - Workspace switching
  - Feature toggling
  - Desktop management
- **Functions Needed**:
  - `rofi-menu <options>`: Show menu
  - `rofi-select <prompt> <items>`: Selection dialog
  - `rofi-input <prompt>`: Text input
- **Use Cases**:
  - `bsp layout select`: Choose layout profile
  - `bsp desktop list`: Interactive desktop switcher
  - `bsp feature toggle`: Enable/disable features

**_ui** (NOT currently in use - MEDIUM PRIORITY)
- **Why It's Needed**: Terminal UI for status/debugging
- **How It Would Be Used**:
  - Pretty-print window tree
  - Status dashboards
  - Interactive debugger
- **Functions Needed**:
  - `ui-table <data>`: Format table
  - `ui-tree <data>`: Format tree
  - `ui-status <items>`: Status indicators
  - `ui-progress <percent>`: Progress bar
- **Use Cases**:
  - `bsp tree`: Visualize window tree
  - `bsp status`: Show feature status
  - `bsp debug`: Interactive debugger

**_notify** (NOT currently in use - LOW PRIORITY)
- **Why It's Needed**: User notifications
- **How It Would Be Used**:
  - Desktop creation/removal notifications
  - Layout change notifications
  - Error notifications
- **Functions Needed**:
  - `notify-send <title> <message>`: Send notification
  - `notify-error <message>`: Error notification
  - `notify-info <message>`: Info notification
- **Use Cases**:
  - Notify when new desktop added
  - Notify on layout profile switch
  - Notify on errors

#### Query & Manipulation Abstractions

**_bspwm** (DOES NOT EXIST - NEW LIBRARY NEEDED - HIGH PRIORITY)
- **Why It's Needed**: High-level bspwm abstractions
- **How It Would Be Used**:
  - Wrapper around bspc commands
  - Type-safe queries
  - Error handling
  - Retry logic
- **Functions Needed**:
  ```zsh
  # Query functions
  bspwm-query-nodes [selector]      # Get node IDs
  bspwm-query-desktops [selector]   # Get desktop IDs
  bspwm-query-monitors [selector]   # Get monitor IDs
  bspwm-query-focused-node          # Get focused node
  bspwm-query-focused-desktop       # Get focused desktop
  bspwm-query-focused-monitor       # Get focused monitor

  # Node manipulation
  bspwm-node-set-state <node> <state>        # Set node state
  bspwm-node-set-flag <node> <flag> <value>  # Set node flag
  bspwm-node-set-border <node> <width>       # Set border width
  bspwm-node-set-color <node> <color>        # Set border color
  bspwm-node-move <node> <desktop>           # Move to desktop
  bspwm-node-swap <node1> <node2>            # Swap nodes
  bspwm-node-focus <node>                    # Focus node
  bspwm-node-close <node>                    # Close node
  bspwm-node-kill <node>                     # Kill node
  bspwm-node-balance <node>                  # Balance subtree

  # Desktop manipulation
  bspwm-desktop-add <monitor> <name>         # Add desktop
  bspwm-desktop-remove <desktop>             # Remove desktop
  bspwm-desktop-rename <desktop> <name>      # Rename desktop
  bspwm-desktop-focus <desktop>              # Focus desktop
  bspwm-desktop-swap <desktop1> <desktop2>   # Swap desktops

  # Monitor manipulation
  bspwm-monitor-add <name>                   # Add monitor
  bspwm-monitor-remove <monitor>             # Remove monitor
  bspwm-monitor-focus <monitor>              # Focus monitor
  bspwm-monitor-rename <monitor> <name>      # Rename monitor

  # Configuration
  bspwm-config-get <key>                     # Get config value
  bspwm-config-set <key> <value>             # Set config value
  bspwm-config-set-node <node> <key> <value> # Set node-specific config

  # Event subscription (wrapper around bspc subscribe)
  bspwm-subscribe <events...>                # Subscribe to events

  # Window properties (wrapper around xprop)
  bspwm-window-class <window>                # Get WM_CLASS
  bspwm-window-name <window>                 # Get WM_NAME
  bspwm-window-command <window>              # Get WM_COMMAND
  bspwm-window-pid <window>                  # Get _NET_WM_PID

  # Utility functions
  bspwm-is-running                           # Check if bspwm is running
  bspwm-version                              # Get bspwm version
  bspwm-validate-selector <selector>         # Validate selector syntax
  ```
- **Integration Benefits**:
  - Centralized bspwm interaction
  - Consistent error handling
  - Easier testing (mock the library)
  - Better documentation
  - Type safety (validate selectors)

**_layout** (DOES NOT EXIST - NEW LIBRARY NEEDED - MEDIUM PRIORITY)
- **Why It's Needed**: Layout profile management
- **How It Would Be Used**:
  - Define layout templates
  - Apply layouts to desktops
  - Save/restore layouts
  - Layout presets (dev, media, work)
- **Functions Needed**:
  ```zsh
  layout-list                        # List available layouts
  layout-save <name>                 # Save current layout
  layout-load <name>                 # Load layout
  layout-apply <desktop> <layout>    # Apply to desktop
  layout-delete <name>               # Delete layout
  layout-export <name> <file>        # Export to file
  layout-import <file>               # Import from file
  ```
- **Layout Definition**:
  ```json
  {
    "name": "dev",
    "description": "Development layout with editor and terminals",
    "tree": {
      "split": "vertical",
      "ratio": 0.618,
      "left": { "type": "window", "class": "Code" },
      "right": {
        "split": "horizontal",
        "ratio": 0.5,
        "top": { "type": "window", "class": "Alacritty" },
        "bottom": { "type": "window", "class": "Alacritty" }
      }
    }
  }
  ```
- **Integration Benefits**:
  - Reproducible layouts
  - Quick workspace setup
  - Layout sharing

**_workspace** (DOES NOT EXIST - NEW LIBRARY NEEDED - LOW PRIORITY)
- **Why It's Needed**: Higher-level workspace abstraction
- **How It Would Be Used**:
  - Named workspaces (not just numbers)
  - Workspace templates
  - Application grouping
  - Auto-launch apps on workspace switch
- **Functions Needed**:
  ```zsh
  workspace-create <name> <template>  # Create workspace
  workspace-switch <name>             # Switch workspace
  workspace-list                      # List workspaces
  workspace-delete <name>             # Delete workspace
  workspace-rename <old> <new>        # Rename workspace
  ```
- **Integration Benefits**:
  - Semantic workspace names
  - Project-based workflows

#### Testing & Validation

**_test** (NOT currently in use - HIGH PRIORITY for development)
- **Why It's Needed**: Testing framework for bsp utilities
- **How It Would Be Used**:
  - Unit tests for logic
  - Integration tests with mock bspc
  - Regression testing
- **Functions Needed**:
  - `test-suite <name>`: Define test suite
  - `test-case <name> <function>`: Define test
  - `test-assert-equal <expected> <actual>`: Assertion
  - `test-mock-command <command> <output>`: Mock external commands
  - `test-run`: Run all tests
- **Integration Benefits**:
  - Confidence in refactoring
  - Prevent regressions
  - Documentation via tests

### Library Enhancement Opportunities

#### _events Enhancements Needed
Current _events library likely doesn't have bspwm-specific features:
- **Add**: Event parsing for bspc subscribe output
- **Add**: Event type validation
- **Add**: Event filtering by selector patterns
- **Add**: Event batching for performance

#### _config Enhancements Needed
- **Add**: JSON schema validation
- **Add**: Config file watching (auto-reload)
- **Add**: Config migration (version upgrades)
- **Add**: Profile management

#### _cache Enhancements Needed
- **Add**: Event-based invalidation
- **Add**: Cache statistics
- **Add**: Persistent cache (disk-backed)

#### _lifecycle Enhancements Needed
- **Add**: Dependency management (feature A requires feature B)
- **Add**: Feature auto-restart on config change
- **Add**: Health checks with auto-recovery

### New Library Opportunities

#### 1. _bspwm (HIGH PRIORITY)
**Purpose**: Complete bspwm/bspc abstraction layer

**Reusability**: HIGH - Any dotfiles using bspwm would benefit
- Other window management utilities
- Status bar modules (show window count, layout, etc.)
- Keybinding scripts
- Automation scripts

**Functionality** (see detailed functions above):
- Query abstraction
- Node manipulation
- Desktop management
- Monitor management
- Configuration
- Event subscription
- Window properties
- Validation

**Implementation Complexity**: Medium
- Mostly wrapper functions around bspc
- Error handling for invalid selectors
- Output parsing
- Event stream parsing

#### 2. _layout (MEDIUM PRIORITY)
**Purpose**: Layout profile management and application

**Reusability**: MEDIUM - Useful for bspwm users with complex workflows

**Functionality** (see detailed functions above):
- Layout templates
- Layout saving/loading
- Layout application
- Import/export

**Implementation Complexity**: Medium-High
- JSON schema for layouts
- Tree construction from template
- Node preselection logic
- Application launching

#### 3. _workspace (LOW PRIORITY)
**Purpose**: Higher-level workspace abstraction

**Reusability**: LOW-MEDIUM - Specific to project-based workflows

**Functionality** (see detailed functions above):
- Named workspaces
- Workspace templates
- Application grouping

**Implementation Complexity**: Medium
- Maps to bspwm desktops
- State tracking
- Application launching

#### 4. _x11 (LOW PRIORITY)
**Purpose**: X11 window property utilities

**Reusability**: HIGH - Any X11 utility would benefit

**Functionality**:
```zsh
x11-window-class <window>        # Get WM_CLASS
x11-window-name <window>         # Get WM_NAME
x11-window-command <window>      # Get WM_COMMAND
x11-window-pid <window>          # Get _NET_WM_PID
x11-window-desktop <window>      # Get desktop
x11-window-geometry <window>     # Get geometry
x11-focused-window               # Get focused window
x11-window-list                  # List all windows
```

**Implementation Complexity**: Low
- Wrapper around xprop
- Output parsing

**Note**: Much of this might already be in _bspwm, so this library might not be needed.

---

## Architecture Design Recommendation

### Recommended Pattern: **Unified Daemon + CLI Dispatcher**

Inspired by systemd's approach (single daemon, multiple responsibilities) and git's CLI pattern (subcommands).

**Why This Pattern?**:
1. **Single Event Loop**: One bspc subscribe process, multiple handlers
2. **Shared State**: All features access common state
3. **Coordinated Actions**: Features can interact (e.g., layout + balance)
4. **Resource Efficiency**: One daemon vs 5 separate daemons
5. **Unified Management**: Single systemd service
6. **Better UX**: Single CLI for all operations

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         User Layer                          │
├─────────────────────────────────────────────────────────────┤
│  CLI Commands:                                              │
│  - bsp balance [on|off|status]                             │
│  - bsp desktops [on|off|config]                            │
│  - bsp flag [on|off|config]                                │
│  - bsp floating-borders [on|off]                           │
│  - bsp ventilate [on|off|config]                           │
│  - bsp side <direction>                                     │
│  - bsp shroud [cover|reveal]                               │
│  - bsp fullscreen                                          │
│  - bsp layout [list|save|load|apply]                       │
│  - bsp status                                              │
│  - bsp daemon [start|stop|restart|status]                  │
│  - bsp tree                                                │
│  - bsp config [get|set|edit]                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     CLI Dispatcher Layer                     │
│                     (.local/bin/bsp)                        │
├─────────────────────────────────────────────────────────────┤
│  - Argument parsing                                         │
│  - Subcommand routing                                       │
│  - IPC with daemon (Unix socket)                           │
│  - Direct execution (for non-daemon commands)              │
└─────────────────────────────────────────────────────────────┘
                            │
                ┌───────────┴────────────┐
                ▼                        ▼
┌──────────────────────────┐  ┌─────────────────────────┐
│   Daemon Commands        │  │  Direct Commands        │
│   (IPC to daemon)        │  │  (Execute immediately)  │
├──────────────────────────┤  ├─────────────────────────┤
│ - balance control        │  │ - side <dir>           │
│ - desktops control       │  │ - shroud cover/reveal  │
│ - flag control           │  │ - fullscreen           │
│ - floating-borders       │  │ - layout apply         │
│ - ventilate control      │  │ - tree (query/display) │
│ - daemon control         │  │ - config edit          │
│ - status query           │  └─────────────────────────┘
└──────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│                      Daemon Layer                           │
│                 (.local/libexec/bsp/bspd)                   │
├─────────────────────────────────────────────────────────────┤
│  Main Event Loop:                                           │
│  - bspc subscribe (all events)                             │
│  - Event router (dispatches to feature handlers)           │
│  - IPC server (Unix socket for CLI communication)          │
│  - State manager (shared state across features)            │
│  - Lifecycle manager (start/stop features)                 │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Feature    │  │   Feature    │  │   Feature    │
│   Handlers   │  │   Handlers   │  │   Handlers   │
├──────────────┤  ├──────────────┤  ├──────────────┤
│ Balance      │  │ Desktops     │  │ Flag         │
│ - node_add   │  │ - node_add   │  │ - node_flag  │
│ - node_remove│  │ - node_remove│  │              │
│ - node_state │  │ - node_state │  │              │
│ - node_geom  │  │ - node_geom  │  │              │
└──────────────┘  └──────────────┘  └──────────────┘

┌──────────────┐  ┌──────────────┐
│ FloatBorders │  │ Ventilate    │
│ - node_state │  │ - node_add   │
│              │  │ - node_remove│
└──────────────┘  └──────────────┘
```

### Main Script Structure

**File**: `/home/andronics/.pkgs/bsp/.local/bin/bsp`

```zsh
#!/usr/bin/env zsh
# Main CLI dispatcher

name=$(basename $0)

# Load libraries
source $(which _log)
source $(which _args)
source $(which _config)
source $(which _bspwm)

# Initialize
config-load "${name}"

# Subcommand routing
case "$1" in
    # Daemon management
    daemon)       bsp-daemon "$@" ;;
    status)       bsp-status "$@" ;;

    # Feature control (IPC to daemon)
    balance)      bsp-feature "balance" "$@" ;;
    desktops)     bsp-feature "desktops" "$@" ;;
    flag)         bsp-feature "flag" "$@" ;;
    floating-borders) bsp-feature "floating-borders" "$@" ;;
    ventilate)    bsp-feature "ventilate" "$@" ;;

    # Direct commands (no daemon needed)
    side)         bsp-side "$@" ;;
    shroud)       bsp-shroud "$@" ;;
    fullscreen)   bsp-fullscreen "$@" ;;

    # Layout management
    layout)       bsp-layout "$@" ;;
    workspace)    bsp-workspace "$@" ;;

    # Utilities
    tree)         bsp-tree "$@" ;;
    config)       bsp-config "$@" ;;

    # Help
    -h|--help|help) bsp-help ;;
    -v|--version)   bsp-version ;;

    *)            log-error "Unknown command: $1" ;;
esac
```

### Subcommand Organization

**Directory**: `/home/andronics/.pkgs/bsp/.local/libexec/bsp/`

```
bsp/
├── bin/
│   └── bsp                          # Main dispatcher
└── libexec/
    └── bsp/
        ├── bspd                     # Daemon (main event loop)
        ├── bsp-common              # Shared utilities
        ├── bsp-daemon              # Daemon control (start/stop/restart)
        ├── bsp-status              # Status reporting
        ├── bsp-feature             # Feature control (enable/disable)
        ├── bsp-side                # Side placement logic
        ├── bsp-shroud              # Shroud cover/reveal
        ├── bsp-fullscreen          # Fullscreen toggle
        ├── bsp-layout              # Layout management
        ├── bsp-workspace           # Workspace management
        ├── bsp-tree                # Tree visualization
        ├── bsp-config              # Config management
        ├── bsp-help                # Help system
        ├── bsp-version             # Version info
        └── features/
            ├── balance.zsh         # Balance feature
            ├── desktops.zsh        # Desktops feature
            ├── flag.zsh            # Flag feature
            ├── floating-borders.zsh # Floating borders feature
            └── ventilate.zsh       # Ventilate feature
```

### Daemon Implementation

**File**: `/home/andronics/.pkgs/bsp/.local/libexec/bsp/bspd`

```zsh
#!/usr/bin/env zsh
# Main daemon process

name="bspd"

# Load libraries
source $(which _log)
source $(which _singleton)
source $(which _onexit)
source $(which _events)
source $(which _lifecycle)
source $(which _config)
source $(which _cache)
source $(which _bspwm)

# Singleton enforcement
_singleton_terminate && [[ $? -eq 1 ]] && exit 1

# Configuration
config-load "bsp"

# State
declare -A FEATURES        # Feature instances
declare -A FEATURE_STATUS  # Feature enabled/disabled

# Load features
_load_features() {
    local feature_dir="${0:h}/features"

    for feature_file in ${feature_dir}/*.zsh; do
        feature_name=$(basename ${feature_file} .zsh)

        log-info "Loading feature: ${feature_name}"
        source ${feature_file}

        FEATURES[${feature_name}]="${feature_name}"
        FEATURE_STATUS[${feature_name}]=$(config-get "features.${feature_name}.enabled" "false")
    done
}

# Initialize features
_init_features() {
    for feature in ${(k)FEATURES}; do
        if [[ ${FEATURE_STATUS[${feature}]} == "true" ]]; then
            log-info "Initializing feature: ${feature}"
            lifecycle-start ${feature}
        fi
    done
}

# Event router
_route_event() {
    local event_line="$1"
    local event_type=$(echo ${event_line} | awk '{print $1}')

    log-debug "Event: ${event_line}"

    # Emit to event bus (handlers will subscribe)
    event-emit ${event_type} "${event_line}"
}

# IPC server
_ipc_server() {
    local socket="${XDG_RUNTIME_DIR}/bsp.sock"

    # Remove old socket
    [[ -S ${socket} ]] && rm ${socket}

    # Create socket with socat/nc
    # Listen for commands from CLI
    # (Implementation depends on IPC mechanism chosen)
}

# Main loop
_main() {
    log-info "Starting bsp daemon"

    # Load and initialize
    _load_features
    _init_features

    # Start IPC server in background
    _ipc_server &

    # Subscribe to all events
    bspwm-subscribe node_add node_remove node_state node_flag node_geometry \
                    desktop_add desktop_remove desktop_focus \
                    monitor_add monitor_remove monitor_focus | \
    while read -r event_line; do
        _route_event "${event_line}"
    done
}

# Cleanup on exit
_cleanup() {
    log-info "Shutting down bsp daemon"

    # Stop all features
    for feature in ${(k)FEATURES}; do
        lifecycle-stop ${feature}
    done

    # Remove socket
    [[ -S ${XDG_RUNTIME_DIR}/bsp.sock ]] && rm ${XDG_RUNTIME_DIR}/bsp.sock
}

onexit-register _cleanup

# Run
_main "$@"
```

### Feature Module Example

**File**: `/home/andronics/.pkgs/bsp/.local/libexec/bsp/features/balance.zsh`

```zsh
# Balance feature - auto-balance window tree

balance_start() {
    log-info "[balance] Starting"

    # Subscribe to relevant events
    event-subscribe "node_add" balance_handle_event
    event-subscribe "node_remove" balance_handle_event
    event-subscribe "node_state" balance_handle_event
    event-subscribe "node_geometry" balance_handle_event
}

balance_stop() {
    log-info "[balance] Stopping"

    # Unsubscribe
    event-unsubscribe balance_handle_event
}

balance_handle_event() {
    local event_line="$1"

    log-debug "[balance] Event: ${event_line}"

    # Balance all windows on desktop
    local windows=$(bspwm-query-nodes ".window")
    for window in ${windows}; do
        bspwm-node-balance "${window}#@north"
    done

    # Balance non-window descendant nodes
    local nodes=$(bspwm-query-nodes "@/ .descendant_of.!window" | grep -v "$(bspwm-query-nodes '@/')")
    for node in ${nodes}; do
        bspwm-node-balance ${node}
    done
}
```

### Library Integration Strategy

#### Phase 1: Core Infrastructure
1. **Use _events** for event bus
   - Single bspc subscribe process
   - Event routing to features
   - Event filtering and debouncing

2. **Use _lifecycle** for feature management
   - Start/stop features dynamically
   - Health checking
   - Auto-recovery

3. **Use _config** for configuration
   - JSON configuration file
   - Profile support
   - Runtime updates

4. **Use _singleton** for daemon
   - Prevent multiple daemon instances
   - PID tracking

5. **Use _log** for logging
   - Structured logging
   - Log levels
   - Centralized logs

#### Phase 2: bspwm Abstraction
1. **Create _bspwm library**
   - Query wrappers
   - Node manipulation
   - Desktop management
   - Monitor management
   - Event subscription
   - Window properties

2. **Use _cache** for performance
   - Cache query results
   - Event-based invalidation
   - TTL expiration

#### Phase 3: Advanced Features
1. **Create _layout library**
   - Layout templates
   - Layout saving/loading
   - Layout application

2. **Use _rofi** for UI
   - Interactive layout selection
   - Feature toggling
   - Desktop switching

3. **Use _ui** for visualization
   - Tree visualization
   - Status dashboard
   - Debug views

4. **Use _notify** for notifications
   - Desktop events
   - Layout changes
   - Errors

### UI/UX Design

#### CLI Interface

**Consistent Patterns**:
- All commands follow `bsp <subcommand> [options] [args]`
- Global flags: `--help`, `--version`, `--debug`, `--config <file>`
- Feature controls: `<feature> [on|off|status|config]`
- Action commands: `<action> [args]`

**Examples**:
```bash
# Daemon management
bsp daemon start                    # Start daemon
bsp daemon stop                     # Stop daemon
bsp daemon restart                  # Restart daemon
bsp daemon status                   # Check status

# Feature control
bsp balance on                      # Enable balance
bsp balance off                     # Disable balance
bsp balance status                  # Check if enabled
bsp balance config                  # Show config

bsp desktops on                     # Enable auto-desktops
bsp desktops config start_id 5      # Set starting ID

bsp ventilate on                    # Enable swallowing
bsp ventilate config add-terminal "kitty"
bsp ventilate config add-no-inhale "zoom"

# Direct actions
bsp side west                       # Place window west
bsp shroud cover                    # Hide all
bsp shroud reveal                   # Show all
bsp fullscreen                      # Toggle fullscreen

# Layout management
bsp layout list                     # List layouts
bsp layout save dev                 # Save current as "dev"
bsp layout load dev                 # Load "dev" layout
bsp layout apply focused dev        # Apply to focused desktop

# Workspace management
bsp workspace create work           # Create workspace
bsp workspace switch work           # Switch to workspace
bsp workspace list                  # List workspaces

# Utilities
bsp tree                            # Show window tree
bsp tree --focused                  # Show focused desktop tree
bsp status                          # Show all feature status
bsp status --json                   # JSON output
bsp config get features.balance.enabled
bsp config set features.balance.enabled true
bsp config edit                     # Open in editor

# Help
bsp help                            # General help
bsp help balance                    # Help for balance
bsp --version                       # Version info
```

#### Rofi Integration

**Interactive Menus**:
```bash
# Layout selection
bsp layout select                   # Rofi menu of layouts

# Workspace switching
bsp workspace select                # Rofi menu of workspaces

# Feature toggling
bsp feature toggle                  # Rofi menu to enable/disable features

# Desktop management
bsp desktop select                  # Rofi menu to switch desktops
```

#### Notification Strategy

**When to Notify**:
- Desktop created/removed (if configured)
- Layout applied
- Feature enabled/disabled
- Errors (always)

**Notification Levels**:
- Info: Desktop changes, layout changes
- Warning: Feature failures, degraded operation
- Error: Critical failures, config errors

#### Status Display Format

**Terminal Output**:
```bash
$ bsp status

BSP Window Manager Utilities
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Daemon: RUNNING (PID 12345)
Uptime: 2 hours 34 minutes

Features:
  ✓ balance           ENABLED
  ✓ desktops          ENABLED  (start_id: 1)
  ✓ flag              ENABLED
  ✓ floating-borders  ENABLED
  ✓ ventilate         ENABLED  (3 terminals, 6 no-inhale)

Windows: 12
Desktops: 5 (2 occupied)
Monitors: 2

Layouts:
  - default
  - dev
  - media
```

**JSON Output** (for scripting):
```json
{
  "daemon": {
    "status": "running",
    "pid": 12345,
    "uptime": 9240
  },
  "features": {
    "balance": { "enabled": true },
    "desktops": { "enabled": true, "config": { "start_id": 1 } },
    "flag": { "enabled": true },
    "floating-borders": { "enabled": true },
    "ventilate": { "enabled": true, "config": { "terminals": 3, "no_inhale": 6 } }
  },
  "windows": 12,
  "desktops": 5,
  "desktops_occupied": 2,
  "monitors": 2,
  "layouts": ["default", "dev", "media"]
}
```

---

## Creative Feature Opportunities

### Advanced Features

#### 1. **Layout Profiles**
- **What**: Pre-defined window layouts (dev, media, work, focus)
- **How**: JSON templates defining tree structure and applications
- **Value**: Instant workspace setup, reproducible environments
- **Automation**: Auto-apply layout when specific app launches

#### 2. **Smart Desktop Management**
- **What**: Semantic desktop names, auto-naming based on apps
- **How**: Monitor launched apps, assign desktop names
- **Value**: "Browser" instead of "Desktop 3"
- **Enhancement**: Workspace persistence across reboots

#### 3. **Window History & Restore**
- **What**: Track window positions, restore after crashes
- **How**: Event logging, state snapshots
- **Value**: Session recovery, undo/redo for window operations
- **Integration**: Save/restore with layouts

#### 4. **Focus Follows Activity**
- **What**: Auto-focus window with recent activity (notifications, output)
- **How**: Monitor window properties, detect changes
- **Value**: Reduce manual focusing for background tasks
- **Enhancement**: Smart focus for urgent windows (alerts, errors)

#### 5. **Dynamic Tiling Rules**
- **What**: Context-aware tiling behavior
- **How**: Rules based on time, desktop, monitor, app count
- **Value**: Different tiling ratios for different scenarios
- **Example**:
  - Media apps: 70/30 split (media/controls)
  - Dev: Golden ratio (0.618)
  - Focus mode: Full-screen everything

#### 6. **Monitor Profile Management**
- **What**: Layouts per monitor configuration
- **How**: Detect monitor setup, apply profile
- **Value**: Instant multi-monitor configuration
- **Use Case**: Laptop + external display, docking station

#### 7. **Application Groups**
- **What**: Group related apps, launch together
- **How**: Define groups in config, launch with single command
- **Value**: One-command workspace setup
- **Example**: `bsp group launch dev` opens editor, terminal, browser

#### 8. **Window Tagging & Filtering**
- **What**: Tag windows with labels, filter by tag
- **How**: Metadata tracking, tag-based queries
- **Value**: Organize windows beyond desktops
- **Use Case**: "Work", "Personal", "Temporary"

#### 9. **Automatic Window Placement**
- **What**: Rules for where windows should open
- **How**: Match window class/name, apply placement rules
- **Value**: No manual window management
- **Example**:
  - Terminals always on Desktop 1
  - Browsers on Desktop 2
  - Media players fullscreen on Desktop 3

#### 10. **Smart Balance**
- **What**: Balance only when beneficial (not always)
- **How**: Heuristics for when balancing improves layout
- **Value**: Reduce unnecessary balancing
- **Enhancement**: Different balance strategies (equal, golden ratio, adaptive)

#### 11. **Desktop Snapshots**
- **What**: Save current desktop state, restore later
- **How**: Capture window list, positions, states
- **Value**: Quick layout switching
- **Use Case**: Save "deep work" setup, restore after interruption

#### 12. **Window Swapping Intelligence**
- **What**: Smart window swapping based on content type
- **How**: Detect window content (video, text, images), suggest swaps
- **Value**: Optimal visual arrangement
- **Example**: Put video on larger split

#### 13. **Ventilate Profiles**
- **What**: Different swallow behaviors per context
- **How**: Multiple no_inhale/terminals lists, switchable
- **Value**: Different workflows (dev vs casual use)
- **Example**: "Work" profile never swallows browser, "Home" profile does

#### 14. **Desktop Workflow Automation**
- **What**: Automate common desktop transitions
- **How**: Trigger-action rules
- **Value**: Reduce manual desktop management
- **Examples**:
  - When last window closes, switch to previous desktop
  - When new window opens on empty desktop, rename desktop
  - When monitor disconnected, migrate windows

#### 15. **Integration with External Tools**
- **What**: Hook into other window management tools
- **How**: Event emission, D-Bus interface
- **Value**: Ecosystem integration
- **Examples**:
  - Update status bar on desktop change
  - Sync with time tracker (window focus = task)
  - Integrate with screenshot tools

### Automation Opportunities

#### 1. **Time-Based Automation**
- Auto-apply "focus" layout during work hours
- Enable "do not disturb" mode (shroud) at specific times
- Switch to "media" layout in evenings

#### 2. **App-Launch Automation**
- Auto-create desktop when specific app launches
- Auto-apply layout when IDE opens
- Auto-enable ventilate when terminal-heavy workflow detected

#### 3. **Context-Aware Behavior**
- Different balance ratios based on screen resolution
- Disable balance for gaming desktops
- Enable floating-borders only on primary monitor

#### 4. **Smart Defaults**
- Auto-detect terminal emulators for ventilate config
- Auto-populate no_inhale list from system apps
- Auto-adjust desktop count based on usage patterns

#### 5. **Predictive Features**
- Learn common window arrangements, suggest layouts
- Predict next desktop switch, preload
- Suggest window groupings based on usage

### Integration Possibilities

#### 1. **Polybar Integration**
- Show active features in bar
- Desktop names in bar
- Window count per desktop
- Current layout name
- Click to toggle features

#### 2. **Dunst/Notification Integration**
- Notify on desktop changes
- Notify on layout apply
- Notify on feature enable/disable
- Show window tree on notification

#### 3. **Rofi Integration**
- Layout selector (with preview)
- Desktop switcher (with window list)
- Feature toggler
- Window search & focus

#### 4. **Pywal Integration**
- Adjust border colors based on colorscheme
- Update flag colors dynamically
- Theme-aware notifications

#### 5. **Status Bar Integration** (lemonbar, i3bar, waybar)
- Module showing bsp status
- Interactive feature toggles
- Layout indicator

#### 6. **Compositor Integration** (picom, compton)
- Sync animations with layout changes
- Fade effects on shroud cover/reveal
- Blur floating windows (but not tiled)

#### 7. **Keybinding Daemon Integration** (sxhkd, xbindkeys)
- Generate keybindings from config
- Dynamic keybinding based on context
- Show keybinding hints in UI

#### 8. **Screen Recording Integration**
- Auto-enable shroud during recording
- Auto-apply "presentation" layout
- Hide distractions

#### 9. **VPN Integration** (via _vpn library)
- Apply "secure" layout when VPN connects
- Move sensitive apps to specific desktop
- Hide windows when VPN disconnects

#### 10. **Time Tracking Integration** (wakatime, timewarrior)
- Track time per desktop/layout
- Auto-tag time based on active layout
- Report productivity by workspace

### UI/UX Improvements

#### 1. **Visual Tree Display**
- ASCII art window tree
- Color-coded by state (tiled, floating, fullscreen)
- Interactive (click to focus)
- Real-time updates

#### 2. **Interactive Configuration**
- TUI for config editing (using _ui)
- Immediate preview of changes
- Validation with helpful errors
- Config templates

#### 3. **Dashboard**
- Overview of all features
- Quick toggles
- Resource usage
- Event log

#### 4. **Better Error Messages**
- Human-readable errors
- Suggestions for fixes
- Links to documentation
- Error codes for debugging

#### 5. **Help System**
- Comprehensive `bsp help`
- Man pages
- Interactive tutorials
- Example workflows

#### 6. **Tab Completion**
- Zsh completion for all commands
- Bash completion
- Context-aware suggestions

#### 7. **Progress Indicators**
- Show progress for long operations (layout apply)
- Spinner for daemon start
- Percentage for bulk operations

#### 8. **Undo/Redo**
- Undo last window operation
- Redo window operation
- Operation history
- Checkpoint/restore

#### 9. **Window Preview**
- Show window thumbnails in tree
- Preview layout before applying
- Show desktop contents in switcher

#### 10. **Accessibility**
- Screen reader support
- High contrast mode
- Keyboard-only navigation
- Configurable colors

---

## Implementation Plan

### Phase 1: Foundation (Week 1)

**Goal**: Core infrastructure and basic daemon

**Tasks**:
1. **Create _bspwm library** (3 days)
   - Query wrappers (bspwm-query-*)
   - Node manipulation (bspwm-node-*)
   - Desktop management (bspwm-desktop-*)
   - Event subscription (bspwm-subscribe)
   - Window properties (bspwm-window-*)
   - Tests (mock bspc)

2. **Create main dispatcher** (1 day)
   - `bsp` main script
   - Argument parsing
   - Subcommand routing
   - Help system
   - Version info

3. **Create daemon foundation** (2 days)
   - `bspd` main daemon
   - Event loop with bspc subscribe
   - Event routing to features
   - Feature loading system
   - Lifecycle management
   - Configuration loading

4. **Migrate one feature** (1 day)
   - Port bsp-balance to new architecture
   - Implement as feature module
   - Test event handling
   - Verify functionality

**Deliverables**:
- Working _bspwm library
- Main bsp dispatcher
- Basic daemon with one feature (balance)
- Test suite for _bspwm

**Success Criteria**:
- [ ] `bsp --help` shows help
- [ ] `bsp daemon start` starts daemon
- [ ] `bsp balance on` enables balance
- [ ] Balance feature works as before
- [ ] Daemon logs to standard location

### Phase 2: Feature Migration (Week 2)

**Goal**: Migrate all existing features to new architecture

**Tasks**:
1. **Migrate event-driven features** (3 days)
   - bsp-desktops → features/desktops.zsh
   - bsp-flag → features/flag.zsh
   - bsp-floating-borders → features/floating-borders.zsh
   - bsp-ventilate → features/ventilate.zsh
   - Test each feature
   - Verify backward compatibility

2. **Migrate CLI utilities** (2 days)
   - bsp-side → bsp-side subcommand
   - bsp-shroud → bsp-shroud subcommand
   - bsp-fullscreen → bsp-fullscreen subcommand
   - Preserve exact behavior

3. **Configuration system** (2 days)
   - JSON config schema
   - Config loading
   - Config validation
   - Default values
   - Migrate ventilate configs to JSON

**Deliverables**:
- All features migrated
- JSON configuration system
- Feature enable/disable working
- Config migration script

**Success Criteria**:
- [ ] All 8 utilities work as before
- [ ] Single daemon replaces 5 separate daemons
- [ ] Config loaded from JSON
- [ ] Features can be toggled at runtime
- [ ] Backward compatibility maintained

### Phase 3: Enhanced Features (Week 3)

**Goal**: Add new features and improvements

**Tasks**:
1. **IPC system** (2 days)
   - Unix socket server in daemon
   - CLI client for IPC
   - Commands: status, feature control
   - Response serialization (JSON)

2. **Status and monitoring** (1 day)
   - `bsp status` command
   - Feature status display
   - Daemon health check
   - JSON output option

3. **Tree visualization** (1 day)
   - `bsp tree` command
   - ASCII art tree
   - Color coding
   - Detailed node info

4. **Improve existing features** (2 days)
   - Implement missing flag handlers (floating, hidden, sticky, etc.)
   - Smart balance (don't balance unnecessarily)
   - Configurable balance strategies
   - Enhanced ventilate (better class detection)
   - Desktop naming

5. **Error handling** (1 day)
   - Graceful degradation (bspwm not running)
   - Error recovery
   - Better error messages
   - Input validation

**Deliverables**:
- IPC system working
- Status command
- Tree visualization
- Enhanced feature implementations
- Robust error handling

**Success Criteria**:
- [ ] `bsp status` shows accurate status
- [ ] `bsp tree` shows window tree
- [ ] All flag handlers implemented
- [ ] Features handle errors gracefully
- [ ] Daemon survives bspwm restart

### Phase 4: Advanced Features (Week 4)

**Goal**: Layout management and advanced automation

**Tasks**:
1. **Create _layout library** (2 days)
   - Layout schema (JSON)
   - Layout saving
   - Layout loading
   - Layout application
   - Pre-defined layouts (dev, media, work)

2. **Layout commands** (1 day)
   - `bsp layout list`
   - `bsp layout save <name>`
   - `bsp layout load <name>`
   - `bsp layout apply <desktop> <name>`
   - Layout export/import

3. **Rofi integration** (1 day)
   - Layout selector
   - Desktop switcher
   - Feature toggler
   - Use _rofi library

4. **Documentation** (2 days)
   - README with quick start
   - Man pages for commands
   - Configuration guide
   - Architecture documentation
   - Example workflows

5. **Testing and polish** (1 day)
   - Integration tests
   - Performance testing
   - Bug fixes
   - Code cleanup

**Deliverables**:
- Layout management system
- Rofi integration
- Comprehensive documentation
- Test suite
- Polished, production-ready code

**Success Criteria**:
- [ ] Layouts can be saved/loaded
- [ ] Rofi menus work
- [ ] Documentation complete
- [ ] Tests passing
- [ ] No known bugs

### Phase 5: Polish & Release (Bonus Week)

**Goal**: Production-ready release

**Tasks**:
1. **Systemd integration** (1 day)
   - Single bsp.service
   - Auto-start on login
   - Proper shutdown
   - Service status integration

2. **Migration guide** (1 day)
   - Migration script (old → new)
   - Backward compatibility shims
   - Deprecation notices
   - Breaking changes documentation

3. **Performance optimization** (1 day)
   - Event debouncing
   - Query caching
   - Reduce bspc calls
   - Benchmark and profile

4. **UI polish** (1 day)
   - Better status display
   - Color output
   - Progress indicators
   - Tab completion

5. **Additional features** (1 day)
   - Monitor profile management
   - Application groups
   - Window tagging
   - (Pick highest-value features)

**Deliverables**:
- Systemd service
- Migration script
- Optimized performance
- Polished UI
- Release-ready package

**Success Criteria**:
- [ ] Systemd service works
- [ ] Migration from old version smooth
- [ ] Performance acceptable
- [ ] UI polished
- [ ] Ready for users

### Effort Estimate

**Total**: **4-5 weeks** (can extend to 5 weeks if adding more advanced features)

**Breakdown**:
- Phase 1 (Foundation): 7 days
- Phase 2 (Migration): 7 days
- Phase 3 (Enhanced): 7 days
- Phase 4 (Advanced): 7 days
- Phase 5 (Polish): 5 days (optional/bonus)

**Confidence Level**: **High** (75%)

**Variables**:
- _bspwm library complexity (might take longer than estimated)
- IPC implementation (depends on chosen mechanism)
- Testing complexity (requires X11 environment setup)
- Feature scope creep (many exciting features possible)

**Assumptions**:
- Full-time work (8 hours/day)
- Existing libraries (_events, _lifecycle, _config) are functional
- bspwm environment available for testing
- No major blockers or unknowns

---

## Risk Assessment

### Technical Risks

#### Risk 1: Event Handling Complexity
**Description**: Coordinating multiple features reacting to same events could cause race conditions or unexpected interactions.

**Likelihood**: Medium
**Impact**: High

**Mitigation**:
- Use event bus pattern with clear ordering
- Features subscribe to event types, don't share state
- Implement event debouncing/throttling
- Extensive testing with multiple features enabled
- Add event logging for debugging

#### Risk 2: bspwm Dependency
**Description**: bspwm API changes or bugs could break functionality.

**Likelihood**: Low
**Impact**: High

**Mitigation**:
- Abstract all bspwm interaction through _bspwm library
- Version detection and compatibility checks
- Graceful degradation on API changes
- Comprehensive error handling
- Monitor bspwm development

#### Risk 3: IPC Complexity
**Description**: Implementing reliable IPC between CLI and daemon is non-trivial.

**Likelihood**: Medium
**Impact**: Medium

**Mitigation**:
- Use simple Unix sockets (not D-Bus initially)
- Implement timeout and retry logic
- Fall back to direct mode if daemon unreachable
- Extensive IPC testing
- Consider using socat or nc for simplicity

#### Risk 4: X11 Dependency
**Description**: Testing requires X11 environment, complicating automation.

**Likelihood**: High
**Impact**: Medium

**Mitigation**:
- Use Xvfb for headless testing
- Mock bspc commands in unit tests
- Separate testable logic from X11 interaction
- Document manual testing procedures
- Consider containerized test environment

#### Risk 5: State Synchronization
**Description**: Daemon state could desync from actual bspwm state.

**Likelihood**: Medium
**Impact**: Medium

**Mitigation**:
- Minimize stateful caching
- Implement cache invalidation on events
- Periodic state reconciliation
- Query bspwm as source of truth
- Add state validation checks

#### Risk 6: Performance Degradation
**Description**: Single daemon handling all events might be slower than separate daemons.

**Likelihood**: Low
**Impact**: Medium

**Mitigation**:
- Event handling should be minimal
- Use caching to reduce bspc calls
- Profile and benchmark
- Implement event batching
- Optimize hot paths

### Compatibility Risks

#### Risk 1: Breaking User Workflows
**Description**: Changing from separate utilities to unified CLI breaks scripts and keybindings.

**Likelihood**: High
**Impact**: High

**Mitigation**:
- Maintain backward-compatible shims (bsp-balance → bsp balance on)
- Document migration path
- Provide migration script
- Keep old binaries as deprecated wrappers
- Announce breaking changes clearly

#### Risk 2: Configuration Migration
**Description**: Moving from text files to JSON config loses user settings.

**Likelihood**: High
**Impact**: Medium

**Mitigation**:
- Auto-migration script
- Preserve old config format as fallback
- Import old configs to new format
- Document config changes
- Provide config examples

#### Risk 3: Systemd Service Changes
**Description**: Replacing 5 services with 1 requires user intervention.

**Likelihood**: High
**Impact**: Medium

**Mitigation**:
- Provide systemd migration script
- Auto-disable old services
- Auto-enable new service
- Document service changes
- Test on fresh install

#### Risk 4: Library Dependencies
**Description**: Requiring new libraries (_events, _lifecycle, _config) that users might not have.

**Likelihood**: Medium
**Impact**: High

**Mitigation**:
- Bundle required libraries
- Check for library availability on start
- Graceful degradation if library missing
- Document dependencies
- Provide installation script

### User Adoption Risks

#### Risk 1: Learning Curve
**Description**: New CLI interface requires learning new commands.

**Likelihood**: High
**Impact**: Medium

**Mitigation**:
- Comprehensive documentation
- Quick start guide
- Example workflows
- Tab completion
- Helpful error messages
- Interactive tutorial

#### Risk 2: Feature Discovery
**Description**: Users might not discover new features.

**Likelihood**: Medium
**Impact**: Low

**Mitigation**:
- Clear feature documentation
- `bsp help` lists all features
- Release notes highlighting new features
- Example configurations
- Community sharing

#### Risk 3: Migration Friction
**Description**: Users might resist migrating from working setup.

**Likelihood**: Medium
**Impact**: Medium

**Mitigation**:
- Emphasize benefits (performance, features)
- Make migration easy (scripts, docs)
- Maintain backward compatibility
- Provide support channel
- Phase deprecation slowly

#### Risk 4: Debugging Difficulty
**Description**: Unified daemon harder to debug than separate utilities.

**Likelihood**: Medium
**Impact**: Medium

**Mitigation**:
- Comprehensive logging
- `bsp status` shows detailed state
- `bsp daemon status` shows health
- Debug mode with verbose output
- Event logging for troubleshooting
- Clear error messages

### Operational Risks

#### Risk 1: Single Point of Failure
**Description**: Single daemon crash disables all features.

**Likelihood**: Low
**Impact**: High

**Mitigation**:
- Robust error handling
- Feature isolation (one feature crash doesn't kill daemon)
- Systemd auto-restart
- Health monitoring
- Graceful degradation

#### Risk 2: Resource Usage
**Description**: Single daemon might use more resources than needed if only one feature used.

**Likelihood**: Low
**Impact**: Low

**Mitigation**:
- Feature enable/disable at runtime
- Only subscribe to needed events
- Efficient event handling
- Memory profiling
- Lazy loading

#### Risk 3: Upgrade Path
**Description**: Future upgrades might break compatibility.

**Likelihood**: Medium
**Impact**: Medium

**Mitigation**:
- Semantic versioning
- Config version tracking
- Auto-migration on upgrade
- Deprecation warnings
- Changelog documentation

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

### Performance Requirements
- [ ] Event handling latency < 50ms
- [ ] Memory usage < 50MB for daemon
- [ ] CPU usage < 1% idle, < 5% during events
- [ ] Startup time < 1 second
- [ ] No noticeable performance degradation vs old utilities

### Quality Requirements
- [ ] Zero crashes during normal operation
- [ ] Graceful handling of bspwm not running
- [ ] Comprehensive error messages
- [ ] All features have tests
- [ ] Code coverage > 70%

### Documentation Requirements
- [ ] README with installation and quick start
- [ ] Man pages for all commands
- [ ] Configuration guide with examples
- [ ] Architecture documentation
- [ ] Migration guide from old version
- [ ] Example workflows

### User Experience Requirements
- [ ] `--help` for all commands
- [ ] Tab completion (zsh and bash)
- [ ] Backward compatibility shims for old binaries
- [ ] Clear migration path
- [ ] Helpful error messages with suggestions
- [ ] Consistent command naming and structure

### Operational Requirements
- [ ] Systemd service file
- [ ] Auto-start on login (optional)
- [ ] Log rotation
- [ ] Health monitoring
- [ ] Auto-recovery from crashes
- [ ] Config validation

---

## Recommendation

### VERDICT: **PROCEED**

### Justification

The bsp package is an **excellent candidate for a greenfield rewrite**:

**Strong Foundation**:
- Proven functionality (users already rely on it)
- Clear use cases and workflows
- Established patterns (event-driven, bspwm integration)

**Significant Improvements Possible**:
- **5x reduction in daemon processes** (5 → 1)
- **Unified user interface** (8 separate commands → 1 CLI dispatcher)
- **Shared infrastructure** eliminates code duplication
- **Advanced features** not possible in current architecture (layouts, profiles, IPC)
- **Better resource usage** (single event loop, caching)
- **Improved UX** (status, tree, help, config validation)

**Manageable Complexity**:
- Individual features are straightforward
- bspwm API is stable and well-documented
- Library ecosystem exists (_log, _singleton, _xdg already in use)
- Event-driven pattern maps well to new architecture

**High ROI**:
- **4 weeks effort** for **permanent infrastructure**
- Enables future features easily
- Better maintainability
- Improved user experience
- Reusable _bspwm library benefits all bspwm utilities

**Acceptable Risks**:
- All identified risks have clear mitigations
- Backward compatibility can be maintained
- Migration path is straightforward
- No fundamental technical blockers

### Next Steps

1. **Approve rewrite plan** (review this document)
2. **Prioritize features** (which advanced features to include in initial release?)
3. **Set up development environment** (bspwm test instance, Xvfb)
4. **Create _bspwm library** (foundation for everything)
5. **Build basic dispatcher and daemon** (proof of concept)
6. **Migrate features incrementally** (one per day)
7. **Add advanced features** (layouts, IPC, rofi)
8. **Document and test** (comprehensive docs and tests)
9. **Beta testing** (internal dogfooding)
10. **Release v1.0** (production-ready)

### Recommended Approach

**Start Small, Iterate Fast**:
1. **Week 1**: _bspwm library + basic daemon + balance feature
   - Validates architecture
   - Proves event routing works
   - Establishes patterns
2. **Week 2**: Migrate all features
   - Achieves feature parity
   - Validates shared infrastructure
   - Users can switch
3. **Week 3**: Enhanced features + IPC
   - Adds value beyond current implementation
   - Improves UX
   - Enables future features
4. **Week 4**: Layout management + polish
   - Killer feature (layouts)
   - Documentation
   - Production-ready

**Decision Points**:
- After Week 1: Architecture validated? Continue or pivot?
- After Week 2: Feature parity achieved? Release beta?
- After Week 3: User feedback? Adjust priorities?
- After Week 4: Ready for production? Release v1.0?

### Expected Outcomes

**Short-term** (1 month):
- Unified, polished bsp utility
- All existing features working better
- Foundation for future features
- Improved resource usage

**Medium-term** (3 months):
- Advanced features in use (layouts, profiles)
- Community contributions easier
- Other bspwm utilities using _bspwm library
- Established patterns for similar rewrites

**Long-term** (6+ months):
- bsp becomes reference implementation for dotfiles utilities
- _bspwm library widely adopted
- Advanced automation features
- Ecosystem of bsp extensions

---

## Conclusion

The bsp package rewrite represents a **high-value, manageable-risk opportunity** to modernize a collection of useful but fragmented utilities into a cohesive, powerful window management system.

**The rewrite enables**:
- Better user experience
- Advanced features (layouts, profiles, automation)
- Improved maintainability
- Reusable infrastructure (_bspwm library)
- Foundation for future innovation

**The rewrite is justified because**:
- Current architecture limits growth
- Code duplication wastes effort
- Users would benefit significantly
- Effort is reasonable (4 weeks)
- Risks are manageable

**Recommendation**: **Proceed with rewrite, following phased implementation plan.**

Start with Phase 1 (Foundation) as a proof-of-concept, validate the architecture, then proceed with remaining phases.

---

*Analysis completed: 2025-11-07*
*Estimated effort: 4-5 weeks (Moderate Complexity)*
*Recommendation: PROCEED*
