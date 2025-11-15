# Window Management Features

Technical reference for all window management features in A Movement Suite.

## Feature Overview

| Feature | Hotkey | Toggle State | File Location |
|---------|--------|--------------|---------------|
| Window Move | LAlt + RButton | `windowMoveEnabled` | src/Features/WindowMove.ahk2 |
| Width Scaling | LCtrl + RButton | `widthScalingEnabled` | src/Features/WindowScaleWidth.ahk2 |
| Height Scaling | LShift + RButton | `heightScalingEnabled` | src/Features/WindowScaleHeight.ahk2 |
| XY Scaling | LCtrl + LShift + RButton | `xyScalingEnabled` | src/Features/WindowScaleXY.ahk2 |
| Always On Top | LCtrl + MButton | `alwaysOnTopEnabled` | src/Features/WindowAlwaysOnTop.ahk2 |
| Window Roll Up | TBD | `windowRollUpEnabled` | src/Features/WindowRollUp.ahk2 |
| Window Dimmer | TBD | `windowDimmerEnabled` | src/Features/WindowDimmer.ahk2 |
| Window Cascade | RAlt + Up | `windowCascadeEnabled` | src/Features/WindowCascade.ahk2 |
| Virtual Desktop | Win + Alt + Left/Right | `virtualDesktopEnabled` | src/Features/WindowVirtualDesktop.ahk2 |
| Disable Modifications | N/A | N/A | src/Features/DisableWindowModifications.ahk2 |

---

## WindowMove (src/Features/WindowMove.ahk2)

### Description
Move windows with intelligent edge snapping to other windows and screen boundaries. Includes grid-based positioning system.

### Hotkey
`LAlt + RButton` (Left Alt + Right Mouse Button)

### Additional Keys During Movement
- **S key**: Enable/disable snapping during movement
- **Z key**: Display grid positioning menu

### Features
1. **Window Movement**
   - Click and drag to move windows
   - Automatically restores maximized windows
   - Smooth real-time movement

2. **Edge Snapping**
   - Snap to other window edges
   - Snap to screen/monitor edges
   - Configurable snap distance
   - Visual feedback with ToolTip

3. **Grid Positioning**
   - Press Z during movement to show grid
   - Quick layouts: halves, thirds, quarters
   - Monitor-aware positioning

### Global Variables
- `windowMoveEnabled` - Feature toggle state
- `enableWindowSnapping` - Snapping state
- `positioningGui` - Grid GUI instance
- `snapDistance` - Distance threshold for snapping (from GlobalConfig)

### Key Functions

#### Window Validation
```ahk
IsValidWindowForMove(winID) {
    ; Checks if window can be moved
    ; Excludes: system windows, desktop, taskbar
}
```

#### Snapping Algorithm
- Checks all edges of moving window against:
  - Other visible windows
  - Monitor work area boundaries
- Snaps when within `snapDistance` pixels
- Prioritizes closest snap target

### Configuration
- Snap distance adjustable via tray menu → "Window Move Settings"
- Default: 10 pixels
- Persisted in UserConfig

### Known Issues
- Grid positioning may not work correctly on multi-monitor setups
- Some windows may have minimum size restrictions

### Source Reference
`src/Features/WindowMove.ahk2:28-end`

---

## WindowScaleWidth (src/Features/WindowScaleWidth.ahk2)

### Description
Scale window width horizontally by dragging mouse left/right.

### Hotkey
`LCtrl + RButton` (Left Ctrl + Right Mouse Button)

### Behavior
1. Hold LCtrl + Right Mouse Button
2. Move mouse horizontally
3. Window width adjusts in real-time
4. Release to apply

### Global Variables
- `widthScalingEnabled` - Feature toggle state
- `minWindowWidth` - Minimum allowed width (from GlobalConfig)

### Window Validation
Uses `IsValidWindowForResize(winID)` to exclude:
- System windows
- Desktop
- Taskbar
- Windows below minimum dimensions

### Implementation Details
- Maintains window top-left position
- Only modifies width dimension
- Respects minimum window width
- Smooth real-time feedback

### Source Reference
`src/Features/WindowScaleWidth.ahk2`

---

## WindowScaleHeight (src/Features/WindowScaleHeight.ahk2)

### Description
Scale window height vertically by dragging mouse up/down.

### Hotkey
`LShift + RButton` (Left Shift + Right Mouse Button)

### Behavior
1. Hold LShift + Right Mouse Button
2. Move mouse vertically
3. Window height adjusts in real-time
4. Release to apply

### Global Variables
- `heightScalingEnabled` - Feature toggle state
- `minWindowHeight` - Minimum allowed height (from GlobalConfig)

### Window Validation
Uses `IsValidWindowForResize(winID)`

### Implementation Details
- Maintains window top-left position
- Only modifies height dimension
- Respects minimum window height
- Smooth real-time feedback

### Source Reference
`src/Features/WindowScaleHeight.ahk2`

---

## WindowScaleXY (src/Features/WindowScaleXY.ahk2)

### Description
Proportional window scaling with live preview GUI showing new dimensions.

### Hotkey
`LCtrl + LShift + RButton` (Left Ctrl + Left Shift + Right Mouse Button)

### Behavior
1. Hold LCtrl + LShift + Right Mouse Button
2. Move mouse to resize proportionally
3. Preview GUI shows new dimensions
4. Release to apply

### Features
- **Proportional Scaling**: Maintains aspect ratio
- **Live Preview**: Semi-transparent overlay shows new size
- **Visual Feedback**: Preview window with dimension display
- **Center-Based Scaling**: Window center remains fixed (optional)

### Global Variables
- `xyScalingEnabled` - Feature toggle state
- Preview GUI colors from GlobalConfig

### Preview GUI
- Semi-transparent window overlay
- Shows width × height dimensions
- Positioned at new window location
- Auto-cleanup on release

### Known Issues
- **Does not work correctly on multi-monitor setups** (documented in README.md:84)
- Preview may flicker on slower systems

### Source Reference
`src/Features/WindowScaleXY.ahk2`

---

## WindowAlwaysOnTop (src/Features/WindowAlwaysOnTop.ahk2)

### Description
Toggle always-on-top state for windows with visual confirmation.

### Hotkey
`LCtrl + MButton` (Left Ctrl + Middle Mouse Button)

### Behavior
1. Hover over target window
2. Press LCtrl + Middle Mouse Button
3. Window toggles always-on-top state
4. ToolTip confirms new state

### Features
- Visual confirmation (ToolTip)
- Validates window eligibility
- Excludes system windows

### Global Variables
- `alwaysOnTopEnabled` - Feature toggle state

### Window Validation
Uses `IsValidWindowForAlwaysOnTop(winID)`:
- Excludes taskbar, desktop
- Excludes system dialogs
- Checks window existence

### Visual Feedback
```
"Window set to Always On Top" - when enabled
"Window removed from Always On Top" - when disabled
```

### Source Reference
`src/Features/WindowAlwaysOnTop.ahk2`

---

## WindowRollUp (src/Features/WindowRollUp.ahk2)

### Description
Roll up windows to show title bar only, maximizing screen space.

### Hotkey
TBD (To Be Determined)

### Features
- Collapse window to title bar
- Preserve window content
- Toggle to restore full window
- Useful for temporary window parking

### Implementation
- Stores original window height
- Resizes to title bar height only
- Restores on second activation

### Source Reference
`src/Features/WindowRollUp.ahk2`

---

## WindowDimmer (src/Features/WindowDimmer.ahk2)

### Description
Dim inactive windows to improve focus on active window.

### Features
- Automatic dimming of background windows
- Configurable opacity level
- Smooth transitions
- Focus-follows-mouse support

### Implementation
- Monitors window activation events
- Applies transparency to inactive windows
- Restores opacity when window activated

### Global Variables
- `windowDimmerEnabled` - Feature toggle state
- `dimOpacity` - Opacity level for dimmed windows

### Source Reference
`src/Features/WindowDimmer.ahk2`

---

## WindowCascade (src/Features/WindowCascade.ahk2)

### Description
Arrange all windows in a cascading pattern for easy access.

### Hotkey
`RAlt + Up` (Right Alt + Up Arrow)

### Features
- Cascades all visible windows
- Positions windows with offset
- Maintains window sizes
- Multi-monitor aware

### Algorithm
1. Get all visible windows
2. Sort by Z-order or creation time
3. Position each with offset (typically 30x30 pixels)
4. Ensure all title bars visible

### Configuration
- Cascade offset configurable
- Monitor selection

### Source Reference
`src/Features/WindowCascade.ahk2`

---

## WindowVirtualDesktop (src/Features/WindowVirtualDesktop.ahk2)

### Description
Move the active window between virtual desktops using keyboard shortcuts. Provides quick access to organize windows across multiple virtual desktops without switching away from the current desktop.

### Hotkeys
- `Win + Alt + Left` - Move active window to previous virtual desktop
- `Win + Alt + Right` - Move active window to next virtual desktop

### Features
- **Circular Navigation**: Wraps around desktop list (moving left from desktop 1 goes to last desktop)
- **Visual Feedback**: Shows tooltip notification with destination desktop number and window title
- **Automatic Window Handling**: Handles maximized windows and window activation automatically
- **System Window Protection**: Prevents moving system windows (taskbar, desktop, etc.)
- **Error Handling**: Gracefully handles invalid windows with error notifications

### Behavior
1. User presses Win+Alt+Left or Win+Alt+Right
2. System validates active window is eligible for movement
3. Window is moved to adjacent virtual desktop (relative +1 or -1)
4. Tooltip displays success message with desktop number
5. If current window was active, activates another window on the current desktop

### Global Variables
- `virtualDesktopEnabled` - Feature toggle state

### Key Functions

#### MoveWindowToPreviousDesktop()
```ahk
; Moves active window to previous desktop (circular navigation)
; Shows notification with destination desktop number
; Handles errors and invalid windows gracefully
```

#### MoveWindowToNextDesktop()
```ahk
; Moves active window to next desktop (circular navigation)
; Shows notification with destination desktop number
; Handles errors and invalid windows gracefully
```

#### IsValidWindowForVirtualDesktop(hwnd)
```ahk
; Validates window can be moved between desktops
; Excludes: system windows, taskbar, desktop, minimized windows
; Returns: true if window can be moved, false otherwise
```

### VirtualDesktop Library Integration
Uses `VD.ahk2` library functions:
- `VD.MoveWindowToRelativeDesktopNum("A", ±1)` - Core movement function
- Automatically handles:
  - COM interface initialization
  - Windows version detection (Win10/Win11 compatibility)
  - Desktop wrapping/circular navigation
  - Maximized window restoration

### Window Validation
Excluded window classes:
- `Progman` - Desktop
- `WorkerW` - Desktop worker window
- `Shell_TrayWnd` - Taskbar
- `NotifyIconOverflowWindow` - System tray overflow
- `SystemTray_Main` - System tray
- `Windows.UI.Core.CoreWindow` - UWP system UI

Also excludes:
- Minimized windows
- Non-existent/invalid windows

### Visual Feedback
Success notification format:
```
✓ Moved to Desktop <number>
(<window title>)
```

Error notification format:
```
✗ <error message>
(<window title>)
```

Notifications auto-dismiss after 2 seconds.

### Configuration
- No configuration required - uses Windows virtual desktop system settings
- Desktop count determined automatically by Windows
- Toggle feature via tray menu: "Virtual Desktop (#!Left/Right)"

### OS Compatibility
- **Windows 10**: Build 14393+ (Anniversary Update)
- **Windows 11**: All builds
- **Windows Server 2022**: Full support

Automatically detects OS version and uses appropriate COM interfaces.

### Known Issues
- Cannot move UWP apps that are pinned to all desktops
- Some system dialogs may not be movable
- Moving a window doesn't switch the user to that desktop (by design)

### Source Reference
`src/Features/WindowVirtualDesktop.ahk2`

### See Also
- `docs/virtual-desktop-api.md` - VirtualDesktopLib API reference
- `src/VirtualDesktopLib/lib/VD.ahk2` - Core library implementation
- `src/VirtualDesktopLib/lib/WindowManagement.ahk2` - Window movement functions

---

## DisableWindowModifications (src/Features/DisableWindowModifications.ahk2)

### Description
System to disable window modifications for specific windows or window classes.

### Purpose
Prevent accidental modifications to:
- System dialogs
- Critical application windows
- Lock screens
- Specific window classes

### Implementation
- Maintains exclusion list
- Checked by all feature validation functions
- Can be configured per window class or title

### Configuration
```ahk
excludedClasses := ["Shell_TrayWnd", "Progman", "WorkerW", "#32768"]
excludedTitles := ["Task Switching", "Program Manager"]
```

### Source Reference
`src/Features/DisableWindowModifications.ahk2`

---

## Common Feature Patterns

### Feature Module Structure
```ahk
; Global state
global featureEnabled := true

; Hotkey handler
Hotkey::{
    if (!featureEnabled)
        return

    MouseGetPos , , &windowID

    if (!IsValidWindow(windowID))
        return

    ; Feature logic
}

; Toggle function
ToggleFeature(*) {
    global featureEnabled
    featureEnabled := !featureEnabled

    if (featureEnabled)
        A_TrayMenu.Check "Feature Name"
    else
        A_TrayMenu.Uncheck "Feature Name"
}
```

### Window Validation Pattern
```ahk
IsValidWindowFor*(winID) {
    if (!WinExist("ahk_id " winID))
        return false

    winClass := WinGetClass("ahk_id " winID)

    excludedClasses := ["Shell_TrayWnd", "Progman", "WorkerW"]
    for class in excludedClasses {
        if (winClass = class)
            return false
    }

    return true
}
```

### State Restoration Pattern
```ahk
; Save original state
WinGetPos &origX, &origY, &origWidth, &origHeight, "ahk_id " winID
isMaximized := WinGetMinMax("ahk_id " winID)

; Restore if maximized
if (isMaximized = 1) {
    WinRestore "ahk_id " winID
    Sleep 10  ; Ensure restore completes
}

; Perform operations...

; Optionally restore original state if needed
```

---

## Feature Interaction Matrix

| Feature | Compatible With | Conflicts With | Notes |
|---------|----------------|----------------|-------|
| WindowMove | All scaling features | - | Can move while resizing |
| Width Scaling | Height Scaling, WindowMove | XY Scaling | Independent dimensions |
| Height Scaling | Width Scaling, WindowMove | XY Scaling | Independent dimensions |
| XY Scaling | WindowMove | Width/Height Scaling | Proportional only |
| Always On Top | All | - | Independent state |
| Window Roll Up | All except scaling | Scaling features | Can't scale rolled window |
| Window Dimmer | All | - | Visual effect only |
| Window Cascade | All | - | One-time action |

---

## Performance Considerations

### Window Detection
- Use cached window IDs when possible
- Validate windows before expensive operations
- Minimize WinGet* calls in tight loops

### Visual Feedback
- Limit ToolTip duration
- Clean up GUI objects promptly
- Use timers for auto-cleanup

### Multi-Monitor
- Cache monitor information
- Use monitor-specific coordinates
- Test thoroughly on different configurations

---

## Adding a New Feature

### Step-by-Step Guide

1. **Create Feature File**
   ```ahk
   ; src/Features/WindowNewFeature.ahk2
   global newFeatureEnabled := true
   ```

2. **Implement Hotkey Handler**
   ```ahk
   Hotkey::{
       if (!newFeatureEnabled)
           return
       ; Implementation
   }
   ```

3. **Add Toggle Function**
   ```ahk
   ToggleNewFeature(*) {
       global newFeatureEnabled
       newFeatureEnabled := !newFeatureEnabled
       ; Update menu
   }
   ```

4. **Include in Main Script**
   ```ahk
   ; In AWindowMovementSuite.ahk2
   #Include src\Features\WindowNewFeature.ahk2
   ```

5. **Add Tray Menu Entry**
   ```ahk
   A_TrayMenu.Add "New Feature (Hotkey)", ToggleNewFeature
   A_TrayMenu.Check "New Feature (Hotkey)"
   ```

6. **Update Documentation**
   - Add to this file (features.md)
   - Update README.md if user-facing
   - Update architecture.md if architectural changes

---

**See Also:**
- `architecture.md` - System architecture
- `configuration.md` - Configuration system
- `ui-components.md` - GUI components
- `../README.md` - User documentation
