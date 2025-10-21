# Architecture Documentation

## System Overview

A Movement Suite is built on AutoHotkey v2.0 with a modular architecture that separates concerns into distinct subsystems. The application runs as a single-instance background process with a system tray interface.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│          AWindowMovementSuite.ahk2 (Main Entry)             │
│  - System setup (ListLines, KeyHistory, CoordMode)          │
│  - Include management                                       │
│  - Tray menu initialization                                 │
│  - Monitor info display                                     │
└──────────────────┬──────────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌──────────────┐      ┌──────────────────┐
│ Core Libs    │      │ Feature Modules  │
│ (lib/)       │      │ (Features/,      │
│              │      │  MouseHotkeys/)  │
└──────┬───────┘      └────────┬─────────┘
       │                       │
       └───────────┬───────────┘
                   ▼
        ┌──────────────────────┐
        │   Windows OS APIs     │
        │   (Win32, COM)        │
        └──────────────────────┘
```

## Component Architecture

### 1. Entry Point (AWindowMovementSuite.ahk2)

**Responsibilities:**
- Initialize AutoHotkey environment settings
- Load core libraries and configuration
- Include all feature modules
- Set up system tray menu
- Display monitor information on startup

**Startup Sequence:**
1. Set AutoHotkey directives (`#Requires`, `#SingleInstance`, `#WinActivateForce`)
2. Configure performance settings (`ListLines 0`, `KeyHistory 0`, etc.)
3. Include core utilities and configuration
4. Include feature modules
5. Include mouse hotkey system
6. Build tray menu with toggle options
7. Initialize menu check states
8. Display monitor information GUI

**File Location:** `AWindowMovementSuite.ahk2:1-135`

### 2. Core Library Layer (src/lib/)

Provides shared functionality to all other modules.

#### lib/Config/
- **GlobalConfig.ahk2**: System-wide configuration constants
  - Color definitions
  - Snap distance defaults
  - Minimum window dimensions
  - Feature default states

- **UserConfig.ahk2**: User-specific persistent settings
  - Custom snap distances
  - Feature enable/disable preferences
  - Saved positions and preferences

#### lib/Core/
- **CoreUtils.ahk2**: General utility functions
  - Window validation helpers
  - Common calculations
  - String/number utilities

- **ColorUtils.ahk2**: Color manipulation
  - Color format conversions
  - Color constants
  - Transparency handling

- **MoveUtils.ahk2**: Window movement utilities
  - Position calculations
  - Snapping algorithms
  - Multi-monitor handling

- **ToggleUtils.ahk2**: Feature toggle management
  - Enable/disable state management
  - Tray menu synchronization

- **SetupEnvironment.ahk2**: Environment initialization
  - Runtime environment configuration
  - Path setup

#### lib/Helpers/
- **ErrorHandler.ahk2**: Centralized error handling
  - Exception logging
  - User notifications
  - Recovery mechanisms

- **LoggingHelper.ahk2**: Logging infrastructure
  - Debug logging
  - Event tracking
  - Performance monitoring

### 3. Feature Layer (src/Features/)

Independent, toggleable window management features.

**Architecture Pattern:**
```ahk
; Each feature follows this pattern:

; Global state
global featureEnabled := true

; Hotkey handler
Hotkey::{
    if (!featureEnabled)
        return

    ; Get target window
    MouseGetPos , , &windowID
    if (!IsValidWindow(windowID))
        return

    ; Feature logic
    PerformFeatureAction(windowID)
}

; Toggle function for tray menu
ToggleFeature(*) {
    global featureEnabled
    featureEnabled := !featureEnabled
    ; Update tray menu...
}

; Validation function
IsValidWindow(winID) {
    ; Window validation logic
}
```

**Feature Modules:**
- WindowMove.ahk2
- WindowScaleWidth.ahk2
- WindowScaleHeight.ahk2
- WindowScaleXY.ahk2
- WindowAlwaysOnTop.ahk2
- WindowRollUp.ahk2
- WindowDimmer.ahk2
- WindowCascade.ahk2
- DisableWindowModifications.ahk2

See `features.md` for detailed feature documentation.

### 4. Mouse Hotkey Layer (src/MouseHotkeys/)

Context-aware mouse and keyboard shortcut system.

**Architecture:**
```
setting_mouse.ahk2 (Loader)
    ├── hk_basic.ahk2 (Navigation)
    ├── hk_code.ahk2 (Code editor)
    ├── hk_programs.ahk2 (Launchers)
    ├── hk_tab.ahk2 (Tab management)
    ├── hk_wheel.ahk2 (Wheel customization)
    ├── windows_explorer.ahk2 (Explorer)
    └── Startbar.ahk2 (Taskbar)
```

**Pattern:**
- RAlt + Function keys as primary triggers
- Modifier detection (Ctrl, Shift, Alt) for variants
- Context detection via `WinGetProcessName("A")`
- Application-specific behavior switching

See `mouse-hotkeys.md` for detailed documentation.

### 5. UI Layer (src/UI/)

User interface components for dialogs and settings.

**Components:**
- **CustomDialogs.ahk2**: Reusable dialog boxes
  - Confirmation dialogs
  - Input prompts
  - Info displays

- **SettingsGUI.ahk2**: Settings interfaces
  - Feature configuration
  - Snap distance adjustment
  - Preference management

See `ui-components.md` for UI documentation.

### 6. Virtual Desktop Layer (src/VirtualDesktopLib/)

Integration with Windows Virtual Desktop system via COM interfaces.

**Structure:**
```
VirtualDesktopLib/lib/
    ├── VD.ahk2 (Main API)
    ├── ComInterfaces.ahk2 (COM definitions)
    ├── DesktopActions.ahk2 (Desktop operations)
    ├── WindowManagement.ahk2 (Window-desktop binding)
    ├── Utils.ahk2 (Utilities)
    ├── DllCalls.ahk2 (Windows API calls)
    └── Notifications.ahk2 (Event notifications)
```

See `virtual-desktops.md` for technical reference.

## Data Flow

### Window Operation Flow

```
User Input (Hotkey)
    ↓
Feature Module Hotkey Handler
    ↓
Check Feature Enabled State
    ↓
Get Mouse Position & Window ID
    ↓
Validate Window (IsValidWindow*)
    ↓
Read Window Properties (WinGetPos, WinGetMinMax)
    ↓
Calculate New State (using MoveUtils, CoreUtils)
    ↓
Apply Changes (WinMove, WinRestore, etc.)
    ↓
Visual Feedback (ToolTip, GUI preview)
```

### Configuration Flow

```
Startup
    ↓
Load GlobalConfig (defaults)
    ↓
Load UserConfig (overrides)
    ↓
Apply to Runtime State
    ↓

User Changes Setting
    ↓
Update Runtime State
    ↓
Save to UserConfig
    ↓
Persist to Settings File
```

### Mouse Hotkey Flow

```
RAlt + Function Key Press
    ↓
Hotkey Handler
    ↓
Detect Modifiers (Ctrl/Shift/Alt)
    ↓
Get Active Application (WinGetProcessName)
    ↓
Context Switch (Application-specific behavior)
    ↓
Send Appropriate Command
```

## Design Patterns

### 1. Module Pattern
Each feature is self-contained with:
- Global state variable
- Hotkey handler(s)
- Toggle function
- Helper functions
- Minimal dependencies

### 2. Factory Pattern
Used for GUI creation:
- Reusable GUI templates
- Configuration-driven GUI generation
- Consistent styling

### 3. Observer Pattern
- Tray menu observes feature state
- Features notify tray menu of state changes
- Menu checkmarks reflect current state

### 4. Strategy Pattern
Mouse hotkeys use strategy pattern:
- Context detection determines strategy
- Different Send commands per application
- Modifier keys select substrategy

## State Management

### Global State Variables
```ahk
; Feature toggles
global windowMoveEnabled := true
global widthScalingEnabled := true
global heightScalingEnabled := true
; ... etc.

; Configuration
global snapDistance := 10
global minWindowWidth := 100
global minWindowHeight := 100

; Runtime state
global positioningGui := ""
global distraction_free := Map()
```

### Configuration Objects
```ahk
; Global configuration (read-only defaults)
GlobalConfig := {
    SnapDistance: 10,
    Colors: {
        Preview: 0x3399FF,
        Border: 0xFF6600
    }
}

; User configuration (read-write preferences)
UserConfig := {
    CustomSnapDistance: 15,
    EnabledFeatures: ["move", "scale"]
}
```

## Multi-Monitor Support

### Monitor Detection
```ahk
; Get monitor count
monitorCount := MonitorGetCount()

; Get monitor for specific coordinates
MonitorGet(monitorNum, &Left, &Top, &Right, &Bottom)

; Get work area (excluding taskbar)
MonitorGetWorkArea(monitorNum, &WorkLeft, &WorkTop, &WorkRight, &WorkBottom)
```

### Cross-Monitor Operations
- Window snapping works across monitors
- Monitor edge detection for snapping
- Work area boundaries respected
- Known issue: XY scaling on multi-monitor (see README.md:84)

## Event Handling

### Hotkey Registration
```ahk
; Global hotkeys
<!RButton::  ; LAlt + Right Mouse Button

; Conditional hotkeys
#HotIf WinActive("ahk_exe explorer.exe")
RButton::CustomAction()
#HotIf
```

### GUI Events
```ahk
; Button click
buttonCtrl.OnEvent("Click", ButtonClickHandler)

; Window resize
gui.OnEvent("Size", GuiSizeHandler)

; Window close
gui.OnEvent("Close", GuiCloseHandler)
```

## Performance Considerations

### Optimizations
- `ListLines 0` - Disable line logging
- `KeyHistory 0` - Disable key history
- `SetWinDelay -1` - No delay between window operations
- `SetControlDelay -1` - No delay between control operations
- `CoordMode "Mouse", "Screen"` - Direct screen coordinates

### Resource Management
- Single instance enforcement (`#SingleInstance Force`)
- GUI cleanup on destroy
- Tooltip cleanup with timers
- Minimal polling, event-driven architecture

## Error Handling Strategy

### Levels of Error Handling

1. **Silent Fail**: Window validation (return early if invalid)
2. **User Notification**: Feature failures (ToolTip messages)
3. **Error Dialog**: Critical failures (custom dialog)
4. **Logging**: Debug and development (LoggingHelper)

### Example Error Handling
```ahk
try {
    WinMove x, y, width, height, "ahk_id " windowID
}
catch Error as err {
    ToolTip "Failed to move window: " err.Message
    SetTimer () => ToolTip(), -3000
}
```

## Security Considerations

### Window Validation
- Exclude system windows (Shell_TrayWnd, Progman, WorkerW)
- Exclude desktop and taskbar
- Check window existence before operations
- Validate window styles and states

### Safe Operations
- Restore maximized windows before moving
- Respect minimum window dimensions
- Boundary checking for multi-monitor
- Graceful degradation on failures

## Extension Points

### Adding New Features
1. Create new file in `src/Features/`
2. Follow feature module pattern
3. Include in main script
4. Add tray menu entry
5. Update documentation

### Adding New Mouse Hotkeys
1. Choose appropriate `hk_*.ahk2` file
2. Follow RAlt + Function key pattern
3. Add context detection if needed
4. Update documentation

### Extending Configuration
1. Add to GlobalConfig for system defaults
2. Add to UserConfig for user preferences
3. Add settings GUI if user-configurable
4. Update documentation

---

**See Also:**
- `features.md` - Feature module details
- `mouse-hotkeys.md` - Mouse hotkey system
- `configuration.md` - Configuration system
- `ui-components.md` - UI layer
- `virtual-desktops.md` - Virtual desktop integration
