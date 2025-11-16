# Architecture Overview

## Directory Structure at a Glance

```
AMovementSuite/
├── AWindowMovementSuite.ahk2    # Main entry point, tray menu setup
├── src/                          # All source code (@src/claude.md)
│   ├── Features/                 # Window management features
│   ├── MouseHotkeys/             # Mouse hotkey handlers (@src/MouseHotkeys/claude.md)
│   ├── lib/                      # Core utilities, config, helpers
│   ├── UI/                       # GUI components
│   └── VirtualDesktopLib/        # Virtual desktop integration
├── docs/                         # Technical reference documentation
├── settings/                     # User configuration files
└── .claude/                      # Claude Code commands and settings
    └── commands/                 # Reusable slash commands
```

## Core Subsystems

### Features/ - Window Management Modules
Each feature is self-contained and toggleable:
- `WindowMove.ahk2` - Window moving with snapping and grid positioning
- `WindowScaleWidth.ahk2`, `WindowScaleHeight.ahk2`, `WindowScaleXY.ahk2` - Window resizing
- `WindowAlwaysOnTop.ahk2` - Toggle always-on-top state
- `WindowRollUp.ahk2` - Roll up windows to title bar
- `WindowDimmer.ahk2` - Dim inactive windows
- `WindowCascade.ahk2` - Cascade window arrangement
- `WindowVirtualDesktop.ahk2` - Move windows between virtual desktops

See @docs/features.md for technical details.

### lib/ - Core Libraries
- **Config/**: `GlobalConfig.ahk2`, `UserConfig.ahk2` - Configuration management
- **Core/**: `CoreUtils.ahk2`, `ColorUtils.ahk2`, `MoveUtils.ahk2`, `ToggleUtils.ahk2` - Shared utilities
- **Helpers/**: `ErrorHandler.ahk2`, `LoggingHelper.ahk2` - Support functions

See @docs/configuration.md for config system details.

### MouseHotkeys/ - Productivity Shortcuts
Extensive mouse-based hotkey system for navigation, program launching, and system control.

Key files: `hk_basic.ahk2`, `hk_code.ahk2`, `hk_programs.ahk2`, `hk_tab.ahk2`, `hk_wheel.ahk2`, `hk_wezterm.ahk2`

See @src/MouseHotkeys/claude.md and @docs/mouse-hotkeys.md for details.

### UI/ - User Interface Components
- `CustomDialogs.ahk2` - Custom dialog boxes
- `SettingsGUI.ahk2` - Settings interfaces

See @docs/ui-components.md for UI system details.

### VirtualDesktopLib/ - Virtual Desktop Integration
Virtual desktop management library with COM interfaces and desktop actions.

Used by `WindowVirtualDesktop.ahk2` feature for moving windows between desktops.

See `docs/virtual-desktop-api.md` for complete API reference and `docs/virtual-desktops.md` for technical details.

## Module Dependencies

```
AWindowMovementSuite.ahk2 (main)
    ↓
    ├── lib/Config/GlobalConfig.ahk2
    ├── lib/Core/* (CoreUtils, ColorUtils, MoveUtils, ToggleUtils)
    ├── Features/* (all feature modules)
    └── MouseHotkeys/setting_mouse.ahk2
            ↓
            └── MouseHotkeys/hk_*.ahk2 (individual hotkey handlers)
```

## Navigation Guide

**For Development:**
- Adding window features → @docs/features.md and `src/Features/`
- Adding mouse hotkeys → @src/MouseHotkeys/claude.md and @docs/mouse-hotkeys.md
- Modifying configuration → @docs/configuration.md
- Understanding patterns → @docs/claude/code-conventions.md
- UI/GUI changes → @docs/ui-components.md

**For Understanding:**
1. Start with this overview
2. Read @src/claude.md for source organization
3. Dive into specific @docs/*.md for technical details
4. Check subsection claude.md files for focused context
