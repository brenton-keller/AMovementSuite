# How to Use Common Windows APIs with DllCall

## Problem
You need to accomplish tasks like precise window positioning, file operations, monitor information, or cursor tracking that AHK built-ins either don't support or don't expose enough control over.

## Quick Context
Windows provides thousands of API functions for system operations. While AutoHotkey has built-in functions for common tasks (WinMove, FileRead, etc.), sometimes you need capabilities those built-ins don't expose: showing windows without stealing focus, getting work area dimensions, reading files with specific sharing modes, or combining multiple operation flags.

This guide shows you complete, working patterns for the most commonly needed Windows API calls.

## Key Concepts

**When to Use DllCall vs Built-ins**
- **Use built-ins** for standard operations - they're safer, more readable, and handle errors gracefully
- **Use DllCall** when you need specific flags, precise control, or features built-ins don't expose
- **Combine both** - use built-ins for simple operations, DllCall for advanced control

**Error Handling Pattern**
All examples follow this pattern:
1. Call the function and capture return value
2. Check if return indicates failure (usually 0 or NULL)
3. Read `A_LastError` for specific error code
4. Clean up resources (CloseHandle for file handles)

**Resource Management**
Many APIs return handles that must be closed:
- File handles from CreateFile → CloseHandle
- Library handles from LoadLibrary → FreeLibrary
- DC handles from GetDC → ReleaseDC

## Decision Matrix: Built-in vs DllCall

| Task | Built-in | DllCall API | When to Use DllCall |
|------|----------|-------------|---------------------|
| Move/resize window | WinMove | SetWindowPos | Need flags (no-activate, frame-change, Z-order) |
| Get window position | WinGetPos | GetWindowRect | Bypass DetectHiddenWindows setting |
| Get client area | (none) | GetClientRect | Need drawable area dimensions |
| Window title | WinGetTitle | GetWindowText | Need exact character count |
| Make topmost | WinSetAlwaysOnTop | SetWindowPos | Temporary topmost, complex flag combos |
| Show/hide window | WinShow/WinHide | ShowWindow | Specific show states (SW_SHOWNOACTIVATE) |
| Read file | FileRead | CreateFile+ReadFile | Need share modes, byte-level control |
| Mouse position | MouseGetPos | GetCursorPos | Need screen coordinates only |
| Monitor info | MonitorGet | GetMonitorInfo | Need work area, precise bounds |
| Window flash | (none) | FlashWindowEx | Get user attention, taskbar notification |

## Pattern 1: Window Manipulation - SetWindowPos

### Complete Function with All Common Operations

```ahk
; SetWindowPos - Precise window positioning with flags
; More powerful than WinMove for advanced scenarios
class WindowAPI {
    ; Constants
    static HWND_TOP := 0
    static HWND_BOTTOM := 1
    static HWND_TOPMOST := -1
    static HWND_NOTOPMOST := -2

    static SWP_NOSIZE := 0x0001
    static SWP_NOMOVE := 0x0002
    static SWP_NOZORDER := 0x0004
    static SWP_NOREDRAW := 0x0008
    static SWP_NOACTIVATE := 0x0010
    static SWP_FRAMECHANGED := 0x0020
    static SWP_SHOWWINDOW := 0x0040
    static SWP_HIDEWINDOW := 0x0080

    ; Show window without activating (no focus steal)
    static ShowNoActivate(hwnd) {
        result := DllCall("SetWindowPos",
            "Ptr", hwnd,
            "Ptr", 0,
            "Int", 0,
            "Int", 0,
            "Int", 0,
            "Int", 0,
            "UInt", this.SWP_SHOWWINDOW | this.SWP_NOACTIVATE |
                    this.SWP_NOMOVE | this.SWP_NOSIZE | this.SWP_NOZORDER,
            "Int")

        if (!result) {
            MsgBox "ShowNoActivate failed: " A_LastError
            return false
        }
        return true
    }

    ; Make window always-on-top
    static MakeTopmost(hwnd) {
        result := DllCall("SetWindowPos",
            "Ptr", hwnd,
            "Ptr", this.HWND_TOPMOST,
            "Int", 0,
            "Int", 0,
            "Int", 0,
            "Int", 0,
            "UInt", this.SWP_NOMOVE | this.SWP_NOSIZE,
            "Int")

        if (!result) {
            MsgBox "MakeTopmost failed: " A_LastError
            return false
        }
        return true
    }

    ; Remove always-on-top
    static RemoveTopmost(hwnd) {
        result := DllCall("SetWindowPos",
            "Ptr", hwnd,
            "Ptr", this.HWND_NOTOPMOST,
            "Int", 0,
            "Int", 0,
            "Int", 0,
            "Int", 0,
            "UInt", this.SWP_NOMOVE | this.SWP_NOSIZE,
            "Int")

        if (!result) {
            MsgBox "RemoveTopmost failed: " A_LastError
            return false
        }
        return true
    }

    ; Move and resize in one atomic operation
    static MoveResize(hwnd, x, y, width, height) {
        result := DllCall("SetWindowPos",
            "Ptr", hwnd,
            "Ptr", 0,
            "Int", x,
            "Int", y,
            "Int", width,
            "Int", height,
            "UInt", this.SWP_NOZORDER | this.SWP_NOACTIVATE,
            "Int")

        if (!result) {
            MsgBox "MoveResize failed: " A_LastError
            return false
        }
        return true
    }
}

; Usage Examples
hwnd := WinExist("A")

; Show notification window without stealing focus
WindowAPI.ShowNoActivate(hwnd)

; Make window stay on top
WindowAPI.MakeTopmost(hwnd)

; Position window precisely
WindowAPI.MoveResize(hwnd, 100, 100, 800, 600)

; Remove topmost status
WindowAPI.RemoveTopmost(hwnd)
```

### Common Mistake: Using WinMove When You Need Flags
```ahk
; PROBLEM: WinShow steals focus from current window
WinShow("ahk_id " hwnd)  ; User loses focus on what they were doing

; SOLUTION: Use SetWindowPos with SWP_NOACTIVATE
WindowAPI.ShowNoActivate(hwnd)  ; Shows window without focus steal
```

## Pattern 2: File Operations - CreateFile and ReadFile

### Complete File Reading with Share Modes

```ahk
; Read file even if another process has it open
ReadFileWithSharing(filename) {
    ; Constants
    GENERIC_READ := 0x80000000
    FILE_SHARE_READ := 0x1
    FILE_SHARE_WRITE := 0x2
    OPEN_EXISTING := 3
    FILE_ATTRIBUTE_NORMAL := 0x80
    INVALID_HANDLE_VALUE := -1

    ; Open file with read+write sharing
    hFile := DllCall("CreateFileW",
        "WStr", filename,
        "UInt", GENERIC_READ,
        "UInt", FILE_SHARE_READ | FILE_SHARE_WRITE,
        "Ptr", 0,              ; Default security
        "UInt", OPEN_EXISTING,
        "UInt", FILE_ATTRIBUTE_NORMAL,
        "Ptr", 0,
        "Ptr")

    if (!hFile || hFile == INVALID_HANDLE_VALUE) {
        error := A_LastError
        errorMsg := ""

        switch error {
            case 2: errorMsg := "File not found"
            case 3: errorMsg := "Path not found"
            case 5: errorMsg := "Access denied"
            case 32: errorMsg := "File in use (sharing violation)"
            default: errorMsg := "Error code: " error
        }

        MsgBox "Cannot open file: " filename "`n" errorMsg
        return ""
    }

    ; Get file size
    fileSizeLow := 0
    fileSizeHigh := 0
    fileSizeLow := DllCall("GetFileSize", "Ptr", hFile, "UInt*", &fileSizeHigh, "UInt")
    fileSize := fileSizeLow + (fileSizeHigh << 32)

    if (fileSize == 0) {
        DllCall("CloseHandle", "Ptr", hFile)
        return ""
    }

    ; Allocate buffer
    buffer := Buffer(fileSize, 0)
    bytesRead := 0

    ; Read file
    result := DllCall("ReadFile",
        "Ptr", hFile,
        "Ptr", buffer,
        "UInt", fileSize,
        "UInt*", &bytesRead,
        "Ptr", 0,
        "Int")

    ; ALWAYS close handle
    DllCall("CloseHandle", "Ptr", hFile)

    if (!result) {
        MsgBox "ReadFile failed: " A_LastError
        return ""
    }

    ; Convert to string (assumes UTF-8)
    return StrGet(buffer, bytesRead, "UTF-8")
}

; Usage
content := ReadFileWithSharing("C:\logs\app.log")
if (content != "")
    MsgBox "File content:`n" SubStr(content, 1, 200)
```

### When to Use CreateFile vs FileRead

**Use FileRead when:**
- Reading entire file into string
- File isn't locked by other processes
- UTF-8/UTF-16 encoding with BOM
- Simple, one-shot read

**Use CreateFile+ReadFile when:**
- File might be open in another program
- Need specific sharing modes (FILE_SHARE_READ/WRITE)
- Reading specific byte ranges
- Reading binary data
- Need to keep file open for multiple operations

### Common Mistake: Forgetting CloseHandle
```ahk
; WRONG - handle leak
ReadFileBad(filename) {
    hFile := DllCall("CreateFileW", "WStr", filename, ..., "Ptr")
    ; ... read operations ...
    return content  ; LEAK: Never closed handle
}

; CORRECT - always clean up
ReadFileGood(filename) {
    hFile := DllCall("CreateFileW", "WStr", filename, ..., "Ptr")

    try {
        ; ... read operations ...
        return content
    } finally {
        DllCall("CloseHandle", "Ptr", hFile)  ; Closes even if error
    }
}
```

## Pattern 3: Monitor Info - GetMonitorInfo

### Complete Monitor Work Area Function

```ahk
; Get monitor's work area (screen minus taskbar)
GetMonitorWorkArea(hwnd := 0) {
    ; Constants
    MONITOR_DEFAULTTONULL := 0x0
    MONITOR_DEFAULTTOPRIMARY := 0x1
    MONITOR_DEFAULTTONEAREST := 0x2
    MONITORINFOF_PRIMARY := 0x1

    ; Get monitor handle
    if (hwnd == 0) {
        ; Get primary monitor
        hMonitor := DllCall("MonitorFromPoint",
            "Int64", 0,  ; pt.x and pt.y both 0
            "UInt", MONITOR_DEFAULTTOPRIMARY,
            "Ptr")
    } else {
        ; Get monitor containing window
        hMonitor := DllCall("MonitorFromWindow",
            "Ptr", hwnd,
            "UInt", MONITOR_DEFAULTTONEAREST,
            "Ptr")
    }

    if (!hMonitor) {
        MsgBox "MonitorFromWindow/Point failed"
        return false
    }

    ; Allocate MONITORINFO structure
    ; typedef struct {
    ;     DWORD cbSize;      // Offset 0, 4 bytes
    ;     RECT  rcMonitor;   // Offset 4, 16 bytes (full monitor)
    ;     RECT  rcWork;      // Offset 20, 16 bytes (work area)
    ;     DWORD dwFlags;     // Offset 36, 4 bytes
    ; } MONITORINFO;         // Total: 40 bytes

    info := Buffer(40, 0)
    NumPut("UInt", 40, info, 0)  ; CRITICAL: Must set cbSize

    result := DllCall("GetMonitorInfoW",
        "Ptr", hMonitor,
        "Ptr", info,
        "Int")

    if (!result) {
        MsgBox "GetMonitorInfo failed: " A_LastError
        return false
    }

    ; Extract both monitor and work area
    return {
        ; Full monitor area (offset 4-19)
        monitorX: NumGet(info, 4, "Int"),
        monitorY: NumGet(info, 8, "Int"),
        monitorWidth: NumGet(info, 12, "Int") - NumGet(info, 4, "Int"),
        monitorHeight: NumGet(info, 16, "Int") - NumGet(info, 8, "Int"),

        ; Work area (offset 20-35) - excludes taskbar
        workX: NumGet(info, 20, "Int"),
        workY: NumGet(info, 24, "Int"),
        workWidth: NumGet(info, 28, "Int") - NumGet(info, 20, "Int"),
        workHeight: NumGet(info, 32, "Int") - NumGet(info, 24, "Int"),

        ; Flags
        isPrimary: NumGet(info, 36, "UInt") & MONITORINFOF_PRIMARY
    }
}

; Center window in monitor's work area
CenterWindowInWorkArea(hwnd) {
    work := GetMonitorWorkArea(hwnd)
    if (!work)
        return false

    ; Get window size
    WinGetPos(&x, &y, &width, &height, "ahk_id " hwnd)

    ; Calculate centered position
    newX := work.workX + (work.workWidth - width) // 2
    newY := work.workY + (work.workHeight - height) // 2

    ; Move window
    WinMove(newX, newY, , , "ahk_id " hwnd)
    return true
}

; Usage examples
hwnd := WinExist("A")

; Get monitor info
info := GetMonitorWorkArea(hwnd)
MsgBox "Work area: " info.workWidth "x" info.workHeight
    . "`nFull monitor: " info.monitorWidth "x" info.monitorHeight
    . "`nPrimary: " (info.isPrimary ? "Yes" : "No")

; Center window
CenterWindowInWorkArea(hwnd)
```

### When to Use GetMonitorInfo vs MonitorGet

**Use MonitorGet when:**
- Need basic monitor dimensions
- Working within AHK script context
- Simple coordinate calculations

**Use GetMonitorInfo when:**
- Need work area specifically (excludes taskbar)
- Need to know if monitor is primary
- Building window managers
- Precise multi-monitor layouts

## Pattern 4: Mouse/Cursor - GetCursorPos

### Complete Mouse Position Function

```ahk
; Get cursor position (screen coordinates)
GetMousePos() {
    ; POINT structure: two LONGs (8 bytes total)
    ; Offset 0: x (4 bytes)
    ; Offset 4: y (4 bytes)

    pt := Buffer(8, 0)

    result := DllCall("GetCursorPos", "Ptr", pt, "Int")

    if (!result) {
        MsgBox "GetCursorPos failed: " A_LastError
        return false
    }

    return {
        x: NumGet(pt, 0, "Int"),
        y: NumGet(pt, 4, "Int")
    }
}

; Get mouse position relative to window
GetMousePosRelativeToWindow(hwnd) {
    ; Get screen coordinates
    screen := GetMousePos()
    if (!screen)
        return false

    ; Get window position
    WinGetPos(&winX, &winY, , , "ahk_id " hwnd)

    return {
        x: screen.x - winX,
        y: screen.y - winY
    }
}

; Usage
pos := GetMousePos()
MsgBox "Mouse at: " pos.x ", " pos.y

hwnd := WinExist("A")
rel := GetMousePosRelativeToWindow(hwnd)
MsgBox "Relative to window: " rel.x ", " rel.y
```

### When to Use GetCursorPos vs MouseGetPos

**Use MouseGetPos when:**
- Need window/control under cursor
- Need relative coordinates to window
- Working with AHK's coordinate modes

**Use GetCursorPos when:**
- Only need screen coordinates
- Building low-level mouse hooks
- Maximum performance (no AHK overhead)

## Pattern 5: Window Info - GetWindowText and GetWindowRect

### Complete Window Information Functions

```ahk
; Get window title (more control than WinGetTitle)
GetWindowTitle(hwnd) {
    ; Get required length
    length := DllCall("GetWindowTextLengthW", "Ptr", hwnd, "Int")

    if (length == 0) {
        error := A_LastError
        if (error != 0) {
            MsgBox "GetWindowTextLength failed: " error
            return ""
        }
        return ""  ; Window has no title
    }

    ; Allocate buffer (characters + 1 for null, × 2 for UTF-16)
    buffer := Buffer((length + 1) * 2, 0)

    chars := DllCall("GetWindowTextW",
        "Ptr", hwnd,
        "Ptr", buffer,
        "Int", length + 1,
        "Int")

    if (chars == 0 && A_LastError != 0) {
        MsgBox "GetWindowText failed: " A_LastError
        return ""
    }

    return StrGet(buffer, "UTF-16")
}

; Get window rectangle (screen coordinates)
GetWindowRect(hwnd) {
    ; RECT structure: 4 LONGs (16 bytes)
    ; Offset 0:  left
    ; Offset 4:  top
    ; Offset 8:  right
    ; Offset 12: bottom

    rect := Buffer(16, 0)

    result := DllCall("GetWindowRect", "Ptr", hwnd, "Ptr", rect, "Int")

    if (!result) {
        MsgBox "GetWindowRect failed: " A_LastError
        return false
    }

    left := NumGet(rect, 0, "Int")
    top := NumGet(rect, 4, "Int")
    right := NumGet(rect, 8, "Int")
    bottom := NumGet(rect, 12, "Int")

    return {
        x: left,
        y: top,
        width: right - left,
        height: bottom - top,
        right: right,
        bottom: bottom
    }
}

; Get client area (drawable region inside window borders)
GetClientRect(hwnd) {
    ; RECT structure: 4 LONGs (16 bytes)
    ; For client rect, left and top are always 0

    rect := Buffer(16, 0)

    result := DllCall("GetClientRect", "Ptr", hwnd, "Ptr", rect, "Int")

    if (!result) {
        MsgBox "GetClientRect failed: " A_LastError
        return false
    }

    return {
        width: NumGet(rect, 8, "Int"),   ; right (left is 0)
        height: NumGet(rect, 12, "Int")  ; bottom (top is 0)
    }
}

; Complete window information
GetCompleteWindowInfo(hwnd) {
    title := GetWindowTitle(hwnd)
    outer := GetWindowRect(hwnd)
    client := GetClientRect(hwnd)

    if (!outer || !client)
        return false

    return {
        title: title,
        x: outer.x,
        y: outer.y,
        width: outer.width,
        height: outer.height,
        clientWidth: client.width,
        clientHeight: client.height,
        borderWidth: (outer.width - client.width) // 2,
        titlebarHeight: outer.height - client.height - ((outer.width - client.width) // 2)
    }
}

; Usage
hwnd := WinExist("A")
info := GetCompleteWindowInfo(hwnd)

MsgBox "Window: " info.title
    . "`nPosition: " info.x ", " info.y
    . "`nOuter: " info.width "x" info.height
    . "`nClient: " info.clientWidth "x" info.clientHeight
    . "`nBorder: " info.borderWidth "px"
    . "`nTitlebar: " info.titlebarHeight "px"
```

### When to Use GetWindowRect vs WinGetPos

**Use WinGetPos when:**
- Standard window operations
- Respect DetectHiddenWindows setting
- Working with AHK window commands

**Use GetWindowRect when:**
- Need to bypass DetectHiddenWindows
- Working with external processes
- Need RECT structure for other APIs
- Building window managers

## Pattern 6: GetClientRect for Drawable Area

### When You Need It
GUI drawing, game overlays, and controls need the client area (inside borders/titlebar).

```ahk
; Position control to fill client area
FillClientArea(hwnd, controlHwnd) {
    client := GetClientRect(hwnd)
    if (!client)
        return false

    ; Resize control to match client area
    DllCall("SetWindowPos",
        "Ptr", controlHwnd,
        "Ptr", 0,
        "Int", 0,
        "Int", 0,
        "Int", client.width,
        "Int", client.height,
        "UInt", 0x0014,  ; SWP_NOZORDER | SWP_NOACTIVATE
        "Int")

    return true
}
```

## Troubleshooting Common Issues

### Issue: Access Denied (Error 5)
**Symptom:** CreateFile returns INVALID_HANDLE_VALUE, A_LastError = 5

**Solutions:**
1. Run script as administrator
2. Check file permissions
3. File might be locked exclusively by another process
4. Use FILE_SHARE_READ | FILE_SHARE_WRITE for shared access

### Issue: Handle is 0 or -1
**Symptom:** CreateFile or other handle-returning function gives invalid handle

**Solution:**
Always check both conditions:
```ahk
hFile := DllCall("CreateFileW", ..., "Ptr")
if (!hFile || hFile == -1) {  ; Check both NULL and INVALID_HANDLE_VALUE
    ; Handle error
}
```

### Issue: GetMonitorInfo Returns Wrong Data
**Symptom:** Coordinates are garbage or zeros

**Solution:**
Always set cbSize field before calling:
```ahk
info := Buffer(40, 0)
NumPut("UInt", 40, info, 0)  ; CRITICAL: Set structure size
DllCall("GetMonitorInfo", "Ptr", hMonitor, "Ptr", info)
```

### Issue: String Shows [object Buffer]
**Symptom:** MsgBox shows "[object Buffer]" instead of text

**Solution:**
Use StrGet to extract string:
```ahk
buffer := Buffer(260 * 2, 0)
DllCall("GetWindowTextW", "Ptr", hwnd, "Ptr", buffer, "Int", 260)
title := StrGet(buffer, "UTF-16")  ; Convert buffer to string
```

### Issue: Works on 32-bit, Fails on 64-bit
**Symptom:** Code works on 32-bit AHK but crashes on 64-bit

**Solution:**
Use Ptr for all handles:
```ahk
; WRONG
hwnd := DllCall("FindWindow", "Str", class, "Str", title, "Int")

; CORRECT
hwnd := DllCall("FindWindow", "Str", class, "Str", title, "Ptr")
```

## Complete Real-World Example: Smart Window Manager

```ahk
; Smart window manager using multiple APIs
class SmartWindowManager {
    ; Show window on specific monitor without focus
    static ShowOnMonitor(hwnd, monitorNum := 1) {
        ; Get target monitor info
        SysGet(&monCount, MonitorCount)
        if (monitorNum > monCount)
            monitorNum := 1

        MonitorGet(monitorNum, &left, &top, &right, &bottom)

        ; Get window size
        rect := GetWindowRect(hwnd)
        if (!rect)
            return false

        ; Position window at monitor center
        x := left + (right - left - rect.width) // 2
        y := top + (bottom - top - rect.height) // 2

        ; Move and show without activating
        DllCall("SetWindowPos",
            "Ptr", hwnd,
            "Ptr", 0,
            "Int", x,
            "Int", y,
            "Int", rect.width,
            "Int", rect.height,
            "UInt", 0x0054,  ; SHOWWINDOW | NOACTIVATE | NOZORDER
            "Int")

        return true
    }

    ; Maximize to work area (excludes taskbar)
    static MaximizeToWorkArea(hwnd) {
        work := GetMonitorWorkArea(hwnd)
        if (!work)
            return false

        DllCall("SetWindowPos",
            "Ptr", hwnd,
            "Ptr", 0,
            "Int", work.workX,
            "Int", work.workY,
            "Int", work.workWidth,
            "Int", work.workHeight,
            "UInt", 0x0014,  ; NOZORDER | NOACTIVATE
            "Int")

        return true
    }
}

; Usage
hwnd := WinExist("A")
SmartWindowManager.ShowOnMonitor(hwnd, 2)  ; Show on second monitor
SmartWindowManager.MaximizeToWorkArea(hwnd)  ; Maximize to work area
```

## Next Steps

You now have working patterns for:
- Window positioning with advanced flags (SetWindowPos)
- File reading with sharing modes (CreateFile, ReadFile)
- Monitor information and work areas (GetMonitorInfo)
- Mouse position tracking (GetCursorPos)
- Window dimensions and client areas (GetWindowRect, GetClientRect)

Practice these patterns, then explore:
- FlashWindowEx for user notifications
- GetWindowThreadProcessId for process information
- EnumWindows for window enumeration
- RegisterHotKey for system-wide hotkeys

Remember: Start with built-ins, reach for DllCall when you need precise control.
