# src/ Directory - Source Code Overview

This directory contains all the source code for A Movement Suite, organized into functional subsystems.

## Directory Structure

```
src/
├── Features/              # Window management features
├── MouseHotkeys/          # Mouse hotkey handlers (see MouseHotkeys/claude.md)
├── lib/                   # Core libraries and utilities
│   ├── Config/            # Configuration management
│   ├── Core/              # Core utilities
│   └── Helpers/           # Helper functions
├── UI/                    # User interface components
├── VirtualDesktopLib/     # Virtual desktop integration
└── calculator.ahk2        # Calculator utility
```

## Subsystem Overview

### Features/ - Window Management Modules

Each feature is self-contained and can be toggled via the tray menu. Features follow a consistent pattern:
- Global `{featureName}Enabled` variable for toggle state
- Hotkey handlers that check enabled state
- Helper functions for window validation
- Integration with core utilities

**Files:**
- `WindowMove.ahk2` - Move windows with snapping, grid positioning
- `WindowScaleWidth.ahk2` - Horizontal window scaling
- `WindowScaleHeight.ahk2` - Vertical window scaling
- `WindowScaleXY.ahk2` - Proportional window scaling with preview
- `WindowAlwaysOnTop.ahk2` - Toggle always-on-top state
- `WindowRollUp.ahk2` - Roll windows up to title bar only
- `WindowDimmer.ahk2` - Dim inactive windows for focus
- `WindowCascade.ahk2` - Cascade window arrangement
- `DisableWindowModifications.ahk2` - Disable modifications for specific windows

See `/docs/features.md` for technical details.

### MouseHotkeys/ - Productivity Shortcuts

Extensive mouse-based hotkey system for navigation, program launching, and system control.
**See `MouseHotkeys/claude.md`** for dedicated documentation on this subsystem.

Key files: `hk_basic.ahk2`, `hk_code.ahk2`, `hk_programs.ahk2`, `hk_tab.ahk2`, `hk_wheel.ahk2`, etc.

See `/docs/mouse-hotkeys.md` for technical reference.

### lib/ - Core Libraries

#### lib/Config/
Configuration management system:
- `GlobalConfig.ahk2` - System-wide defaults, constants, colors
- `UserConfig.ahk2` - User preferences, persistent settings

See `/docs/configuration.md` for details.

#### lib/Core/
Core utility modules shared across features:
- `CoreUtils.ahk2` - General utility functions
- `ColorUtils.ahk2` - Color manipulation and constants
- `MoveUtils.ahk2` - Window movement and positioning utilities
- `ToggleUtils.ahk2` - Feature toggle management
- `SetupEnvironment.ahk2` - Environment initialization

These modules provide reusable functionality to avoid code duplication across features.

#### lib/Helpers/
Support functions:
- `ErrorHandler.ahk2` - Centralized error handling
- `LoggingHelper.ahk2` - Logging infrastructure

### UI/ - User Interface Components

GUI components for user interaction:
- `CustomDialogs.ahk2` - Custom dialog boxes and prompts
- `SettingsGUI.ahk2` - Settings interface (e.g., snap distance configuration)

See `/docs/ui-components.md` for UI system details.

### VirtualDesktopLib/ - Virtual Desktop Integration

Library for Windows virtual desktop management using COM interfaces:
- `lib/VD.ahk2` - Main virtual desktop API
- `lib/ComInterfaces.ahk2` - COM interface definitions
- `lib/DesktopActions.ahk2` - Desktop manipulation actions
- `lib/WindowManagement.ahk2` - Window-to-desktop operations
- `lib/Utils.ahk2`, `lib/DllCalls.ahk2`, `lib/Notifications.ahk2` - Supporting utilities

See `/docs/virtual-desktops.md` for technical reference.

## Module Dependencies

### Dependency Flow
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

### Include Pattern
Features and hotkey modules include core libraries as needed:
```ahk
#Include %A_ScriptDir%\src\lib\Config\GlobalConfig.ahk2
#Include %A_ScriptDir%\src\lib\Core\CoreUtils.ahk2
```

## Common Patterns

### Feature Toggle Pattern
```ahk
global featureEnabled := true  ; Default enabled state

; Hotkey handler checks enabled state
Hotkey::{
    if (!featureEnabled)
        return
    ; Feature logic...
}

; Toggle function for tray menu
ToggleFeature(*) {
    global featureEnabled
    featureEnabled := !featureEnabled
    ; Update tray menu check...
}
```

### Window Validation Pattern
```ahk
IsValidWindowForFeature(winID) {
    ; Check if window exists
    if (!WinExist("ahk_id " winID))
        return false

    ; Exclude system windows, desktop, etc.
    winClass := WinGetClass("ahk_id " winID)
    excludedClasses := ["Shell_TrayWnd", "Progman", "WorkerW"]

    for class in excludedClasses {
        if (winClass = class)
            return false
    }

    return true
}
```

### Mouse Position & Window Detection
```ahk
MouseGetPos &mouseX, &mouseY, &windowID
WinGetPos &winX, &winY, &winWidth, &winHeight, "ahk_id " windowID
```

### Configuration Access
```ahk
; Access global config values
snapDistance := GlobalConfig.SnapDistance
previewColor := GlobalConfig.Colors.Preview
```

## Development Guidelines

### Adding a New Feature
1. Create new file in `Features/` (e.g., `WindowNewFeature.ahk2`)
2. Define global enabled variable: `global newFeatureEnabled := true`
3. Implement hotkey handler(s)
4. Create toggle function: `ToggleNewFeature(*)`
5. Add to main script: `#Include src\Features\WindowNewFeature.ahk2`
6. Add tray menu entry in `AWindowMovementSuite.ahk2`
7. **Update `/docs/features.md`** with technical details

### Adding a New Mouse Hotkey
See `MouseHotkeys/claude.md` for detailed instructions.

### Modifying Core Utilities
1. Identify appropriate module (CoreUtils, ColorUtils, MoveUtils, etc.)
2. Add function with clear documentation
3. Ensure no side effects (pure functions preferred)
4. **Update `/docs/configuration.md`** if adding config options

### UI Changes
1. Modify or create components in `UI/`
2. Follow existing GUI patterns from `SettingsGUI.ahk2`
3. **Update `/docs/ui-components.md`** with changes

## Key Concepts

### Global State Management
- Feature toggle states stored as global variables
- Configuration accessed through GlobalConfig/UserConfig objects
- Avoid excessive global state; prefer parameters

### Hotkey Scope
- Most hotkeys are global (system-wide)
- Some hotkeys are application-specific (use `#HotIf`)
- Mouse hotkeys typically check modifier keys for variant behaviors

### Error Handling
- Use try/catch for potentially failing operations
- Window operations may fail if window closes mid-operation
- Validate windows before manipulation

## Quick File Reference

**Most frequently modified files:**
- `Features/Window*.ahk2` - When adding/modifying window features
- `MouseHotkeys/hk_*.ahk2` - When adding/modifying mouse shortcuts
- `lib/Config/GlobalConfig.ahk2` - When adding new configuration options
- `UI/SettingsGUI.ahk2` - When adding settings interfaces

**Core dependencies:**
- `lib/Core/CoreUtils.ahk2` - Shared utilities used everywhere
- `lib/Core/ToggleUtils.ahk2` - Feature toggle management
- `lib/Config/GlobalConfig.ahk2` - System configuration

---

**Next Steps:**
- For mouse hotkey development → `MouseHotkeys/claude.md`
- For feature details → `/docs/features.md`
- For configuration → `/docs/configuration.md`
- For architecture → `/docs/architecture.md`
