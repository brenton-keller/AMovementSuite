# VirtualDesktop Library API Reference

Comprehensive API documentation for the VirtualDesktopLib (VD.ahk2) - Windows Virtual Desktop management library.

## Overview

The VirtualDesktop library provides a comprehensive interface to Windows Virtual Desktop functionality through COM interfaces. It supports Windows 10 (build 14393+), Windows 11, and Windows Server 2022 with automatic version detection.

**Library Location**: `src/VirtualDesktopLib/lib/VD.ahk2`

**Include Statement**:
```ahk
#Include %A_ScriptDir%\src\VirtualDesktopLib\lib\VD.ahk2
```

## Architecture

The library is organized into functional modules:

```
VD.ahk2 (main container class)
├── ComInterfaces.ahk2      # COM interface definitions & OS detection
├── WindowManagement.ahk2   # Window-to-desktop operations
├── DesktopActions.ahk2     # Desktop navigation & manipulation
├── Utils.ahk2              # Desktop names, GUIDs, utility methods
├── DllCalls.ahk2           # Direct DLL method implementations
└── Notifications.ahk2      # Virtual desktop event notifications
```

All methods are accessible through the `VD` class as static methods.

---

## Window Management Functions

Functions for moving windows between virtual desktops.

### VD.MoveWindowToDesktopNum()

Moves a window to a specific virtual desktop by absolute desktop number.

**Signature**:
```ahk
VD.MoveWindowToDesktopNum(wintitle, desktopNum)
```

**Parameters**:
- `wintitle` (String) - Window identifier (e.g., "ahk_id " windowID, "A" for active window)
- `desktopNum` (Integer) - Target desktop number (1-based)

**Returns**:
- `-1` on error (invalid window or desktop)
- Implicitly void on success

**Behavior**:
- Validates window existence
- If moving active window, activates another window on current desktop first
- Handles maximized windows automatically
- Moves window to specified desktop number

**Example**:
```ahk
; Move active window to desktop 3
VD.MoveWindowToDesktopNum("A", 3)

; Move specific window by ID
windowID := WinGetID("A")
VD.MoveWindowToDesktopNum("ahk_id " windowID, 2)
```

**Source**: `WindowManagement.ahk2:2-28`

---

### VD.MoveWindowToRelativeDesktopNum()

Moves a window by a relative offset from its current desktop. **Best for left/right navigation.**

**Signature**:
```ahk
VD.MoveWindowToRelativeDesktopNum(wintitle, relative_count)
```

**Parameters**:
- `wintitle` (String) - Window identifier
- `relative_count` (Integer) - Relative offset
  - `+1` = next desktop (right)
  - `-1` = previous desktop (left)
  - `+2` = skip one desktop forward
  - etc.

**Returns**:
- Destination desktop number (1-based) on success
- `-1` on error

**Behavior**:
- Calculates absolute desktop from current position + offset
- **Wraps around** (circular navigation)
- Moving left from desktop 1 goes to last desktop
- Moving right from last desktop goes to desktop 1

**Example**:
```ahk
; Move active window to previous desktop
desktopNum := VD.MoveWindowToRelativeDesktopNum("A", -1)

; Move active window to next desktop
desktopNum := VD.MoveWindowToRelativeDesktopNum("A", +1)

; With feedback
desktopNum := VD.MoveWindowToRelativeDesktopNum("A", -1)
if (desktopNum > 0)
    ToolTip("Moved to Desktop " desktopNum)
else
    ToolTip("Failed to move window")
```

**Source**: `WindowManagement.ahk2:29-34`

---

### VD.MoveWindowToCurrentDesktop()

Moves a window from any desktop to the currently active desktop.

**Signature**:
```ahk
VD.MoveWindowToCurrentDesktop(wintitle, activateYourWindow := true)
```

**Parameters**:
- `wintitle` (String) - Window identifier
- `activateYourWindow` (Boolean, optional) - Whether to activate the window after moving (default: true)

**Returns**:
- `-1` on error
- Implicitly void on success

**Behavior**:
- Retrieves current active desktop
- Moves window to that desktop
- Optionally activates the window

**Example**:
```ahk
; Bring window to current desktop and activate it
VD.MoveWindowToCurrentDesktop("ahk_exe notepad.exe")

; Bring window to current desktop without activating
VD.MoveWindowToCurrentDesktop("ahk_exe notepad.exe", false)
```

**Source**: `WindowManagement.ahk2:35-48`

---

### VD.getDesktopNumOfWindow()

Gets the desktop number where a window currently resides.

**Signature**:
```ahk
VD.getDesktopNumOfWindow(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**:
- Desktop number (1-based) where window is located
- `-1` on error (window not found)

**Example**:
```ahk
; Get desktop number of active window
desktopNum := VD.getDesktopNumOfWindow("A")
ToolTip("Active window is on Desktop " desktopNum)

; Check if window is on current desktop
windowDesktop := VD.getDesktopNumOfWindow("ahk_exe chrome.exe")
currentDesktop := VD.getCurrentDesktopNum()
if (windowDesktop = currentDesktop)
    MsgBox("Window is on current desktop")
```

**Source**: `WindowManagement.ahk2:49-57`

---

## Desktop Navigation Functions

Functions for switching between virtual desktops.

### VD.goToDesktopNum()

Switches the user to a specific virtual desktop by absolute number.

**Signature**:
```ahk
VD.goToDesktopNum(desktopNum)
```

**Parameters**:
- `desktopNum` (Integer) - Target desktop number (1-based)

**Returns**: Void

**Example**:
```ahk
; Switch to desktop 2
VD.goToDesktopNum(2)
```

**Source**: `DesktopActions.ahk2`

---

### VD.goToRelativeDesktopNum()

Switches to a desktop by relative offset from current desktop.

**Signature**:
```ahk
VD.goToRelativeDesktopNum(relative_count)
```

**Parameters**:
- `relative_count` (Integer) - Relative offset
  - `+1` = next desktop
  - `-1` = previous desktop

**Returns**: Void

**Behavior**:
- Wraps around (circular navigation)

**Example**:
```ahk
; Go to previous desktop
VD.goToRelativeDesktopNum(-1)

; Go to next desktop
VD.goToRelativeDesktopNum(+1)

; Example hotkeys (from VD_Example.ahk2)
#NumpadSub::VD.goToRelativeDesktopNum(-1)  ; Win+NumpadMinus
#NumpadAdd::VD.goToRelativeDesktopNum(+1)  ; Win+NumpadPlus
```

**Source**: `DesktopActions.ahk2`

---

### VD.getCurrentDesktopNum()

Gets the number of the currently active virtual desktop.

**Signature**:
```ahk
VD.getCurrentDesktopNum()
```

**Parameters**: None

**Returns**:
- Current desktop number (1-based)

**Example**:
```ahk
currentDesktop := VD.getCurrentDesktopNum()
ToolTip("You are on Desktop " currentDesktop)
```

**Source**: `DesktopActions.ahk2:29-33`

---

### VD.getRelativeDesktopNum()

Calculates an absolute desktop number from a relative offset.

**Signature**:
```ahk
VD.getRelativeDesktopNum(anchor_desktopNum, relative_count)
```

**Parameters**:
- `anchor_desktopNum` (Integer) - Starting desktop number (1-based)
- `relative_count` (Integer) - Relative offset

**Returns**:
- Absolute desktop number (1-based)

**Behavior**:
- Handles wrapping/circular navigation
- Used internally by `MoveWindowToRelativeDesktopNum()` and `goToRelativeDesktopNum()`

**Example**:
```ahk
; Calculate what desktop is 2 positions to the right of desktop 5
resultDesktop := VD.getRelativeDesktopNum(5, 2)

; Calculate previous desktop from current
currentDesktop := VD.getCurrentDesktopNum()
previousDesktop := VD.getRelativeDesktopNum(currentDesktop, -1)
```

**Source**: `DesktopActions.ahk2:82-91`

---

## Desktop Information Functions

Functions for querying desktop information.

### VD.getCount()

Gets the total number of virtual desktops.

**Signature**:
```ahk
VD.getCount()
```

**Parameters**: None

**Returns**:
- Total number of virtual desktops (Integer)

**Example**:
```ahk
desktopCount := VD.getCount()
MsgBox("You have " desktopCount " virtual desktops")

; Validate desktop number
desktopNum := 5
if (desktopNum > VD.getCount())
    MsgBox("Desktop " desktopNum " does not exist")
```

**Source**: `DesktopActions.ahk2:2-4`

---

### VD.getDesktopName()

Gets the name of a virtual desktop.

**Signature**:
```ahk
VD.getDesktopName(desktopNum)
```

**Parameters**:
- `desktopNum` (Integer) - Desktop number (1-based)

**Returns**:
- Desktop name (String)
- Empty string if desktop has no custom name

**Example**:
```ahk
; Get name of desktop 2
desktopName := VD.getDesktopName(2)
if (desktopName)
    ToolTip("Desktop 2 is named: " desktopName)
else
    ToolTip("Desktop 2 has no custom name")
```

**Source**: `Utils.ahk2`

---

### VD.setDesktopName()

Sets a custom name for a virtual desktop.

**Signature**:
```ahk
VD.setDesktopName(desktopNum, name)
```

**Parameters**:
- `desktopNum` (Integer) - Desktop number (1-based)
- `name` (String) - Custom name for the desktop

**Returns**: Void

**Example**:
```ahk
; Name desktop 1 as "Work"
VD.setDesktopName(1, "Work")

; Name desktop 2 as "Personal"
VD.setDesktopName(2, "Personal")
```

**Source**: `Utils.ahk2`

---

### VD.goToDesktopByName()

Switches to a desktop by its custom name.

**Signature**:
```ahk
VD.goToDesktopByName(name)
```

**Parameters**:
- `name` (String) - Desktop name to switch to

**Returns**: Void

**Behavior**:
- Case-sensitive name matching
- No-op if desktop with that name doesn't exist

**Example**:
```ahk
; Switch to "Work" desktop
VD.goToDesktopByName("Work")

; Example hotkey (from VD_Example.ahk2)
+NumpadUp::VD.goToDesktopByName("AArrow")
```

**Source**: `Utils.ahk2`

---

## Desktop Creation & Removal Functions

Functions for managing the desktop lifecycle.

### VD.createDesktop()

Creates a new virtual desktop.

**Signature**:
```ahk
VD.createDesktop(name := "")
```

**Parameters**:
- `name` (String, optional) - Custom name for the new desktop

**Returns**:
- Desktop number of newly created desktop (Integer)

**Example**:
```ahk
; Create a new desktop without a name
newDesktopNum := VD.createDesktop()

; Create a new desktop named "Gaming"
gamingDesktop := VD.createDesktop("Gaming")
ToolTip("Created Gaming desktop as #" gamingDesktop)
```

**Source**: `DesktopActions.ahk2`

---

### VD.removeDesktop()

Removes a virtual desktop.

**Signature**:
```ahk
VD.removeDesktop(desktopNum, fallbackDesktopNum := 0)
```

**Parameters**:
- `desktopNum` (Integer) - Desktop to remove (1-based)
- `fallbackDesktopNum` (Integer, optional) - Desktop to switch to if removing current desktop (default: 0 = auto)

**Returns**: Void

**Behavior**:
- Windows on removed desktop are moved to another desktop
- If removing current desktop, switches to fallback desktop
- Cannot remove the only remaining desktop

**Example**:
```ahk
; Remove desktop 3
VD.removeDesktop(3)

; Remove current desktop and switch to desktop 1
currentDesktop := VD.getCurrentDesktopNum()
VD.removeDesktop(currentDesktop, 1)

; Example hotkey (from VD_Example.ahk2)
#NumpadRight::VD.removeDesktop(VD.getCurrentDesktopNum())
```

**Source**: `DesktopActions.ahk2`

---

### VD.MoveDesktop()

Moves a virtual desktop to a new position in the desktop order.

**Signature**:
```ahk
VD.MoveDesktop(sourceDesktopNum, targetDesktopNum)
```

**Parameters**:
- `sourceDesktopNum` (Integer) - Current position of desktop to move (1-based)
- `targetDesktopNum` (Integer) - Target position (1-based)

**Returns**:
- `true` on success
- Throws `ValueError` if indices are out of range
- Throws `Error` if COM call fails

**Behavior**:
- Validates source and target indices are within range
- If source equals target, returns true immediately (no-op)
- Moves desktop from source position to target position
- Other desktops shift accordingly to make room
- Desktop order updates immediately in Windows Task View

**Example**:
```ahk
; Move desktop 4 to position 2
VD.MoveDesktop(4, 2)

; Move newly created desktop to after current
currentNum := VD.getCurrentDesktopNum()
newDesktopNum := VD.getCount()
VD.MoveDesktop(newDesktopNum, currentNum + 1)

; With error handling
try {
    VD.MoveDesktop(5, 2)
    ToolTip("Desktop moved successfully")
} catch ValueError as err {
    MsgBox("Invalid desktop index: " err.Message)
} catch Error as err {
    MsgBox("Move failed: " err.Message)
}
```

**Source**: `DesktopActions.ahk2:136-172`

**Notes**:
- Uses COM method `MoveDesktop` at vtable index 8 (or 9 for Win11 22H2 specific builds)
- Requires Windows 10 build 14393+ or Windows 11
- Changes are immediate and visible in Win+Tab view

---

### VD.CreateDesktopAfterCurrent()

Creates a new virtual desktop immediately after the current desktop and returns to the original desktop.

**Signature**:
```ahk
VD.CreateDesktopAfterCurrent()
```

**Parameters**: None

**Returns**:
- IVirtualDesktop object of the newly created desktop

**Behavior**:
- Gets current desktop position
- Creates new desktop (Windows creates it at the end)
- Moves new desktop to position current+1
- Returns user to original desktop
- Original desktop remains at same position number

**Example**:
```ahk
; Create desktop after current (stay on current)
VD.CreateDesktopAfterCurrent()

; Create and track the new desktop
currentNum := VD.getCurrentDesktopNum()
newDesktop := VD.CreateDesktopAfterCurrent()
; User is still on desktop currentNum
; New desktop is at currentNum + 1

; Example hotkey for Win+Ctrl+D
^#d::{
    currentNum := VD.getCurrentDesktopNum()
    VD.CreateDesktopAfterCurrent()
    ToolTip("Created Desktop " . (currentNum + 1))
}
```

**Source**: `DesktopActions.ahk2:178-203`

**Notes**:
- User stays on original desktop (not switched to new desktop)
- New desktop appears immediately after current in desktop order
- Includes small delays (50ms) to ensure operations complete
- Perfect for Win+Ctrl+D hotkey to insert desktops without disruption

---

## Window Pinning Functions

Functions for pinning windows to appear on all desktops.

### VD.IsPinWindow()

Checks if a window is pinned to all virtual desktops.

**Signature**:
```ahk
VD.IsPinWindow(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**:
- `true` if window is pinned
- `false` if window is not pinned

**Example**:
```ahk
; Check if active window is pinned
if VD.IsPinWindow("A")
    ToolTip("Window is pinned to all desktops")
else
    ToolTip("Window is not pinned")
```

**Source**: `WindowManagement.ahk2`

---

### VD.PinWindow()

Pins a window to appear on all virtual desktops.

**Signature**:
```ahk
VD.PinWindow(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**: Void

**Behavior**:
- Pinned windows appear on every virtual desktop
- Useful for always-visible tools (taskbar, widgets, etc.)

**Example**:
```ahk
; Pin active window
VD.PinWindow("A")

; Pin calculator to all desktops
VD.PinWindow("ahk_exe calc.exe")
```

**Source**: `WindowManagement.ahk2`

---

### VD.UnPinWindow()

Unpins a window so it only appears on its current desktop.

**Signature**:
```ahk
VD.UnPinWindow(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**: Void

**Example**:
```ahk
; Unpin active window
VD.UnPinWindow("A")
```

**Source**: `WindowManagement.ahk2`

---

### VD.TogglePinWindow()

Toggles the pin state of a window.

**Signature**:
```ahk
VD.TogglePinWindow(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**: Void

**Behavior**:
- If pinned, unpins the window
- If not pinned, pins the window

**Example**:
```ahk
; Toggle pin state of window under mouse
#MButton::{
    MouseGetPos ,, &mouseWin
    VD.TogglePinWindow("ahk_id " mouseWin)
}
```

**Source**: `WindowManagement.ahk2`

---

## Application Pinning Functions

Functions for pinning all windows of an application to all desktops.

### VD.IsPinApp()

Checks if an application is pinned (all its windows appear on all desktops).

**Signature**:
```ahk
VD.IsPinApp(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier (any window from the application)

**Returns**:
- `true` if application is pinned
- `false` otherwise

**Example**:
```ahk
; Check if Notepad is pinned
if VD.IsPinApp("ahk_exe notepad.exe")
    ToolTip("Notepad is pinned to all desktops")
```

**Source**: `WindowManagement.ahk2`

---

### VD.PinApp()

Pins an application so all its windows appear on all desktops.

**Signature**:
```ahk
VD.PinApp(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**: Void

**Example**:
```ahk
; Pin all Chrome windows to all desktops
VD.PinApp("ahk_exe chrome.exe")
```

**Source**: `WindowManagement.ahk2`

---

### VD.UnPinApp()

Unpins an application.

**Signature**:
```ahk
VD.UnPinApp(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**: Void

**Example**:
```ahk
; Unpin Chrome
VD.UnPinApp("ahk_exe chrome.exe")
```

**Source**: `WindowManagement.ahk2`

---

### VD.TogglePinApp()

Toggles the pin state of an application.

**Signature**:
```ahk
VD.TogglePinApp(wintitle)
```

**Parameters**:
- `wintitle` (String) - Window identifier

**Returns**: Void

**Example**:
```ahk
; Toggle pin state of application under mouse
^#MButton::{
    MouseGetPos ,, &mouseWin
    VD.TogglePinApp("ahk_id " mouseWin)
}
```

**Source**: `WindowManagement.ahk2`

---

## Usage Examples

### Basic Window Movement

```ahk
; Move active window to previous/next desktop
#!Left::VD.MoveWindowToRelativeDesktopNum("A", -1)
#!Right::VD.MoveWindowToRelativeDesktopNum("A", +1)

; Move active window to specific desktop
#!1::VD.MoveWindowToDesktopNum("A", 1)
#!2::VD.MoveWindowToDesktopNum("A", 2)
#!3::VD.MoveWindowToDesktopNum("A", 3)
```

### Desktop Navigation

```ahk
; Navigate between desktops
#^Left::VD.goToRelativeDesktopNum(-1)
#^Right::VD.goToRelativeDesktopNum(+1)

; Go to specific desktop
#^1::VD.goToDesktopNum(1)
#^2::VD.goToDesktopNum(2)
```

### Desktop Management

```ahk
; Create and name desktops
workDesktop := VD.createDesktop("Work")
personalDesktop := VD.createDesktop("Personal")
gamingDesktop := VD.createDesktop("Gaming")

; Switch between named desktops
^!w::VD.goToDesktopByName("Work")
^!p::VD.goToDesktopByName("Personal")
^!g::VD.goToDesktopByName("Gaming")
```

### Window Pinning

```ahk
; Pin/unpin window under mouse
#MButton::{
    MouseGetPos ,, &mouseWin
    VD.TogglePinWindow("ahk_id " mouseWin)
}

; Always pin taskbar-like applications
VD.PinApp("ahk_exe RocketDock.exe")
VD.PinWindow("ahk_exe Rainmeter.exe")
```

### Query Information

```ahk
; Show desktop information
ShowDesktopInfo() {
    currentDesktop := VD.getCurrentDesktopNum()
    totalDesktops := VD.getCount()
    desktopName := VD.getDesktopName(currentDesktop)

    info := "Desktop " currentDesktop " of " totalDesktops
    if (desktopName)
        info .= " (" desktopName ")"

    ToolTip(info)
    SetTimer(() => ToolTip(), -2000)
}

#^i::ShowDesktopInfo()
```

### Complex Workflow

```ahk
; Move window and follow it
MoveWindowAndFollow(direction) {
    ; Get current desktop and window
    currentDesktop := VD.getCurrentDesktopNum()

    ; Move window
    targetDesktop := VD.MoveWindowToRelativeDesktopNum("A", direction)

    ; Follow to new desktop
    if (targetDesktop > 0)
        VD.goToDesktopNum(targetDesktop)
}

; Win+Shift+Left/Right to move and follow
#!+Left::MoveWindowAndFollow(-1)
#!+Right::MoveWindowAndFollow(+1)
```

---

## Error Handling

Most functions return error indicators:

```ahk
; Check return values
desktopNum := VD.MoveWindowToRelativeDesktopNum("A", -1)
if (desktopNum = -1) {
    MsgBox("Failed to move window")
} else if (desktopNum > 0) {
    ToolTip("Moved to Desktop " desktopNum)
}

; Use try/catch for robustness
try {
    VD.MoveWindowToDesktopNum("A", 5)
} catch Error as err {
    MsgBox("Error: " err.Message)
}
```

---

## OS Compatibility

The library automatically detects Windows version and uses appropriate COM interfaces:

| Windows Version | Build Number | Support Level |
|----------------|--------------|---------------|
| Windows 10 (pre-Anniversary) | < 14393 | Not supported |
| Windows 10 Anniversary Update+ | 14393-20347 | Full support |
| Windows 10 21H2 / Server 2022 | 20348-22000 | Full support |
| Windows 11 (initial) | 22000-22482 | Full support |
| Windows 11 (22H2+) | 22483+ | Full support |

**Detection happens automatically** in `ComInterfaces.ahk2:_init()`.

---

## Performance Considerations

- **COM Initialization**: First call may take 10-50ms for COM interface setup
- **Subsequent Calls**: ~1-5ms per operation (very fast)
- **Window Movement**: Slightly slower (~5-15ms) due to window activation handling
- **Desktop Switching**: ~10-30ms depending on animation settings

**Optimization Tips**:
- Cache desktop numbers instead of calling `getCurrentDesktopNum()` repeatedly
- Batch operations when possible
- Disable desktop switch animations in Windows settings for faster switching

---

## Thread Safety

The library is **not thread-safe**. Design notes:
- Intended for single-threaded hotkey handlers (standard AutoHotkey pattern)
- Avoid concurrent calls from multiple threads
- COM interfaces are apartment-threaded

---

## Known Limitations

1. **UWP Apps**: Some UWP apps may not move correctly
2. **System Windows**: Cannot move taskbar, desktop, or system UI
3. **Pinned Apps**: Cannot move windows that are pinned to all desktops
4. **No Event Callbacks**: Library doesn't provide desktop change notifications (yet)
5. **Windows 10 Pre-Anniversary**: Not supported (build < 14393)

---

## Troubleshooting

### Window Movement Fails

**Symptoms**: `MoveWindowToDesktopNum()` returns -1

**Solutions**:
- Verify window exists: `WinExist(wintitle)`
- Check window class is not excluded (system windows)
- Ensure desktop number is valid (1 to `getCount()`)
- Try with different window identifier

### Desktop Count Always Returns 1

**Symptoms**: `getCount()` always returns 1 even with multiple desktops

**Solutions**:
- Verify virtual desktops are enabled in Windows
- Check Windows build number (must be 14393+)
- Restart script to reinitialize COM interfaces
- Check Task View shows multiple desktops

### COM Interface Errors

**Symptoms**: Errors about COM interfaces or DLL calls

**Solutions**:
- Verify Windows version compatibility
- Run script as administrator (some operations require elevation)
- Restart Windows Explorer (may fix COM registration issues)
- Check Windows Update (ensure latest patches)

---

## See Also

- **Implementation Files**:
  - `src/VirtualDesktopLib/lib/VD.ahk2` - Main container class
  - `src/VirtualDesktopLib/lib/WindowManagement.ahk2` - Window operations
  - `src/VirtualDesktopLib/lib/DesktopActions.ahk2` - Desktop navigation
  - `src/VirtualDesktopLib/lib/Utils.ahk2` - Utility functions

- **Examples**:
  - `src/VirtualDesktopLib/VD_Example.ahk2` - Usage examples

- **Project Documentation**:
  - `docs/features.md` - WindowVirtualDesktop feature documentation
  - `docs/virtual-desktops.md` - General virtual desktop integration docs

- **External Resources**:
  - [Windows Virtual Desktop API](https://docs.microsoft.com/en-us/windows/win32/api/_virtualdesktop/) - Microsoft documentation
  - [IVirtualDesktopManager](https://docs.microsoft.com/en-us/windows/win32/api/shobjidl_core/nn-shobjidl_core-ivirtualdesktopmanager) - COM interface documentation

---

## Credits

Original library implementation based on community reverse-engineering of Windows Virtual Desktop COM interfaces.

AutoHotkey v2.0 port and enhancements by the AMovementSuite project.
