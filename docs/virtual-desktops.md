# Virtual Desktop Library

Technical reference for the Virtual Desktop integration system in A Movement Suite.

## Overview

The VirtualDesktopLib provides Windows Virtual Desktop management capabilities through COM interfaces, enabling:
- Desktop creation, deletion, and switching
- Moving windows between desktops
- Desktop notifications and events
- Multi-desktop window management

**Location:** `src/VirtualDesktopLib/`

## Architecture

```
VirtualDesktopLib/lib/
    ├── VD.ahk2                    # Main API entry point
    ├── ComInterfaces.ahk2         # COM interface definitions
    ├── DesktopActions.ahk2        # Desktop manipulation
    ├── WindowManagement.ahk2      # Window-to-desktop operations
    ├── Utils.ahk2                 # Utility functions
    ├── DllCalls.ahk2              # Windows API calls
    ├── Notifications.ahk2         # Event notifications
    └── VD_Example.ahk2            # Usage examples
```

---

## Module Reference

### VD.ahk2 - Main API

**Purpose:** Primary interface for virtual desktop operations.

**Key Functions:**
- `VD.GetCount()` - Get number of virtual desktops
- `VD.GetCurrent()` - Get current desktop index
- `VD.GetDesktops()` - Get array of all desktops
- `VD.SwitchTo(index)` - Switch to desktop by index
- `VD.Create()` - Create new virtual desktop
- `VD.Remove(index)` - Remove virtual desktop

**Usage Example:**
```ahk
#Include %A_ScriptDir%\src\VirtualDesktopLib\lib\VD.ahk2

; Get desktop count
desktopCount := VD.GetCount()
MsgBox "You have " desktopCount " virtual desktops"

; Switch to desktop 2
VD.SwitchTo(2)

; Create new desktop
VD.Create()

; Get current desktop
currentDesktop := VD.GetCurrent()
```

---

### ComInterfaces.ahk2 - COM Definitions

**Purpose:** Defines COM interface GUIDs and vtable structures for Windows Virtual Desktop APIs.

**Key Components:**
- `IServiceProvider` - Service provider interface
- `IVirtualDesktopManager` - Desktop manager interface
- `IVirtualDesktop` - Individual desktop interface
- `IVirtualDesktopManagerInternal` - Internal desktop operations

**Technical Details:**
- Uses Windows undocumented COM interfaces
- GUID definitions for interface identification
- vtable method offsets for COM calls
- Windows version compatibility handling

**Note:** This module is highly technical and version-dependent. Changes to Windows may break compatibility.

---

### DesktopActions.ahk2 - Desktop Operations

**Purpose:** High-level desktop manipulation functions.

**Key Functions:**
- `CreateDesktop()` - Create new virtual desktop
- `RemoveDesktop(desktop)` - Remove specified desktop
- `SwitchToDesktop(desktop)` - Switch to desktop
- `GetAdjacentDesktop(desktop, direction)` - Get next/previous desktop
- `MoveToDesktop(desktop, direction)` - Move to adjacent desktop

**Usage Example:**
```ahk
; Create new desktop
newDesktop := CreateDesktop()

; Switch to it
SwitchToDesktop(newDesktop)

; Get next desktop
nextDesk := GetAdjacentDesktop(newDesktop, 1)  ; 1 = right, -1 = left

; Move to next
MoveToDesktop(newDesktop, 1)
```

---

### WindowManagement.ahk2 - Window Operations

**Purpose:** Move windows between virtual desktops.

**Key Functions:**
- `MoveWindowToDesktop(windowID, desktop)` - Move window to desktop
- `GetWindowDesktop(windowID)` - Get window's current desktop
- `IsWindowOnCurrentDesktop(windowID)` - Check if window on active desktop
- `MoveWindowToDesktopNumber(windowID, desktopNum)` - Move by index

**Usage Example:**
```ahk
; Get window under mouse
MouseGetPos , , &windowID

; Move to desktop 3
MoveWindowToDesktopNumber(windowID, 3)

; Check which desktop window is on
desktop := GetWindowDesktop(windowID)

; Check if on current desktop
if IsWindowOnCurrentDesktop(windowID)
    MsgBox "Window is on current desktop"
```

---

### Utils.ahk2 - Utility Functions

**Purpose:** Helper functions for virtual desktop operations.

**Key Functions:**
- `GetDesktopCount()` - Wrapper for desktop count
- `GetCurrentDesktopIndex()` - Get active desktop index (0-based)
- `ValidateDesktopIndex(index)` - Validate desktop index
- `GetDesktopName(index)` - Get desktop name (if supported)

---

### DllCalls.ahk2 - Windows API

**Purpose:** Low-level Windows API calls for desktop operations.

**Key Functions:**
- `DwmGetWindowAttribute()` - Get window DWM attributes
- `SetWindowPos()` - Set window position
- `GetDesktopWindow()` - Get desktop window handle
- Helper functions for COM object manipulation

**Technical Details:**
- Direct DllCall to Windows APIs
- Handles structures and pointers
- Platform-specific code (x64/x86)

---

### Notifications.ahk2 - Event System

**Purpose:** Desktop change notifications and event handling.

**Key Features:**
- Desktop switch notifications
- Desktop creation/destruction events
- Window moved events
- Event callback registration

**Usage Example:**
```ahk
; Register callback for desktop switch
VD.OnDesktopSwitch(DesktopSwitched)

DesktopSwitched(oldDesktop, newDesktop) {
    ToolTip "Switched from desktop " oldDesktop " to " newDesktop
    SetTimer () => ToolTip(), -2000
}
```

---

## Common Use Cases

### 1. Move Window to Specific Desktop
```ahk
; Get active window
activeWin := WinGetID("A")

; Move to desktop 2
MoveWindowToDesktopNumber(activeWin, 2)

; Optionally switch to that desktop
VD.SwitchTo(2)
```

### 2. Create Desktop and Move Window
```ahk
; Create new desktop
newDesktop := CreateDesktop()

; Get window under mouse
MouseGetPos , , &windowID

; Move window to new desktop
MoveWindowToDesktop(windowID, newDesktop)

; Switch to new desktop
SwitchToDesktop(newDesktop)
```

### 3. Cycle Through Desktops
```ahk
; Hotkey to go to next desktop
#Right::{  ; Win+Right
    current := VD.GetCurrent()
    count := VD.GetCount()
    next := Mod(current + 1, count)
    VD.SwitchTo(next)
}

; Hotkey to go to previous desktop
#Left::{  ; Win+Left
    current := VD.GetCurrent()
    count := VD.GetCount()
    prev := Mod(current - 1 + count, count)
    VD.SwitchTo(prev)
}
```

### 4. Desktop-Aware Window Management
```ahk
; Only show windows on current desktop
ShowCurrentDesktopWindows() {
    allWindows := WinGetList()

    for windowID in allWindows {
        if IsWindowOnCurrentDesktop(windowID)
            WinShow "ahk_id " windowID
        else
            WinHide "ahk_id " windowID
    }
}
```

---

## Windows Version Compatibility

### Supported Windows Versions
- Windows 10 (Build 10240+)
- Windows 11

### Version-Specific Considerations

**Windows 10:**
- Virtual Desktop COM interfaces introduced in Build 10240
- Interface GUIDs may vary by build
- Some features require specific builds

**Windows 11:**
- Enhanced virtual desktop features
- Desktop naming support
- Improved COM interface stability
- New APIs for desktop wallpapers

### Compatibility Checking
```ahk
CheckVirtualDesktopSupport() {
    ; Check Windows version
    if (A_OSVersion < "10.0")
        return false  ; Windows 10+ required

    ; Try to initialize VD library
    try {
        count := VD.GetCount()
        return true
    }
    catch Error {
        return false
    }
}

if (!CheckVirtualDesktopSupport()) {
    MsgBox "Virtual Desktops not supported on this system"
    ExitApp
}
```

---

## Error Handling

### Common Errors

**1. COM Interface Initialization Failed**
```ahk
try {
    VD.Init()
}
catch Error as err {
    MsgBox "Failed to initialize Virtual Desktop library: " err.Message
}
```

**2. Invalid Desktop Index**
```ahk
desktopIndex := 10  ; May not exist

if (desktopIndex >= 0 && desktopIndex < VD.GetCount()) {
    VD.SwitchTo(desktopIndex)
}
else {
    MsgBox "Desktop index " desktopIndex " does not exist"
}
```

**3. Window Move Failed**
```ahk
try {
    MoveWindowToDesktopNumber(windowID, targetDesktop)
}
catch Error as err {
    MsgBox "Failed to move window: " err.Message
}
```

---

## Performance Considerations

### Caching Desktop Information
```ahk
; Cache desktop count if checking frequently
static desktopCount := 0
static lastUpdate := 0

GetCachedDesktopCount() {
    static desktopCount, lastUpdate

    ; Update cache every 5 seconds
    if (A_TickCount - lastUpdate > 5000) {
        desktopCount := VD.GetCount()
        lastUpdate := A_TickCount
    }

    return desktopCount
}
```

### Minimizing COM Calls
```ahk
; DO: Batch operations
desktops := VD.GetDesktops()
for desktop in desktops {
    ; Process each desktop
}

; DON'T: Repeated calls
Loop VD.GetCount() {
    desktop := VD.GetDesktop(A_Index)  ; Slower
}
```

---

## Integration with A Movement Suite

### Potential Integration Points

**1. Move Windows to Desktop via Grid**
Extend grid positioning system to include "Send to Desktop" options.

**2. Desktop-Aware Window Snapping**
Snap windows only to other windows on the same desktop.

**3. Per-Desktop Feature States**
Enable/disable features based on active desktop.

**4. Desktop Switching Shortcuts**
Add mouse hotkeys for desktop navigation.

**Example Integration:**
```ahk
; In MouseHotkeys - switch desktops
RAlt & F9::{  ; Next desktop
    current := VD.GetCurrent()
    count := VD.GetCount()
    VD.SwitchTo(Mod(current + 1, count))
}

RAlt & F8::{  ; Previous desktop
    current := VD.GetCurrent()
    count := VD.GetCount()
    VD.SwitchTo(Mod(current - 1 + count, count))
}
```

---

## Troubleshooting

### Library Not Working

**Check:**
1. Windows 10/11 version is compatible
2. COM security settings allow automation
3. Windows Virtual Desktop feature is enabled
4. No third-party virtual desktop software conflicts

**Debug:**
```ahk
; Test basic functionality
try {
    count := VD.GetCount()
    MsgBox "Virtual Desktops: " count
}
catch Error as err {
    MsgBox "Error: " err.Message "`n`nStack:`n" err.Stack
}
```

### Desktops Not Switching

**Check:**
1. Desktop index is valid (0 to count-1)
2. Windows animation settings not interfering
3. No permission issues

**Debug:**
```ahk
; Verify desktop exists before switching
targetDesktop := 2
if (targetDesktop >= 0 && targetDesktop < VD.GetCount()) {
    MsgBox "Switching to desktop " targetDesktop
    VD.SwitchTo(targetDesktop)
}
else {
    MsgBox "Desktop " targetDesktop " does not exist"
}
```

### Windows Not Moving

**Check:**
1. Window exists and is valid
2. Target desktop exists
3. Window is not pinned to all desktops
4. Window type supports desktop assignment

**Debug:**
```ahk
; Verify window and desktop
if WinExist("ahk_id " windowID) {
    if (targetDesktop >= 0 && targetDesktop < VD.GetCount()) {
        MsgBox "Moving window " windowID " to desktop " targetDesktop
        MoveWindowToDesktopNumber(windowID, targetDesktop)
    }
}
```

---

## Advanced Topics

### Pinning Windows to All Desktops
Some windows may be pinned to show on all desktops (e.g., Task Manager). These windows cannot be moved.

### Desktop Wallpapers (Windows 11)
Windows 11 supports per-desktop wallpapers. VirtualDesktopLib may expose APIs for this.

### Desktop Names
Windows 11 allows naming virtual desktops. Access through extended VD APIs.

---

**See Also:**
- `VD_Example.ahk2` - Example usage
- `architecture.md` - System integration
- Microsoft Virtual Desktop documentation (limited, mostly undocumented APIs)
