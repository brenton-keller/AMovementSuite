# How-To: Work with Structs in DllCall

## Problem

You need to pass structures (like RECT, POINT, SYSTEMTIME, or MONITORINFO) to Windows API functions, but you're not sure how to allocate memory, calculate offsets, or read/write struct fields correctly.

## Quick Context

**What are structs?** Structures (structs) are contiguous blocks of memory containing multiple fields at fixed byte offsets. When you call Windows API functions, many expect you to pass a pointer to such a structure. The function either reads values from it (input) or fills it with data (output).

**Memory layout basics:** Each field in a struct occupies a specific number of bytes starting at a specific offset from the structure's beginning. For example, a RECT contains four Int values (left, top, right, bottom), each 4 bytes, for a total of 16 bytes at offsets 0, 4, 8, and 12.

**Why this matters:** Getting struct definitions wrong causes crashes, corrupted data, or mysterious failures. You must know the exact size, field types, and offsets for each structure you work with.

## Key Concepts

### Buffer Allocation
In AHK v2, use `Buffer(size, 0)` to allocate memory for structs:
```ahk
rect := Buffer(16, 0)  ; 16 bytes, zero-filled
```
The second parameter (0) zero-fills memory, which is safer than uninitialized memory.

### Field Offsets
Calculate where each field starts by adding up the sizes of previous fields:
```
Field 1 at offset 0
Field 2 at offset 0 + size_of_field_1
Field 3 at offset (0 + size_of_field_1 + size_of_field_2)
```

### NumPut - Writing to Structs
`NumPut(type, value, buffer, offset)` writes a value into a buffer at a specific offset:
```ahk
NumPut("Int", 100, buffer, 0)     ; Write 100 at offset 0
NumPut("Int", 200, buffer, 4)     ; Write 200 at offset 4
```

You can also chain values for sequential writes:
```ahk
NumPut("Int", left, "Int", top, "Int", right, "Int", bottom, buffer)
```

### NumGet - Reading from Structs
`NumGet(buffer, offset, type)` extracts a value from a buffer:
```ahk
left := NumGet(rect, 0, "Int")    ; Read Int from offset 0
top := NumGet(rect, 4, "Int")     ; Read Int from offset 4
```

**Critical:** The type must match what was written or what the API function filled in.

### Alignment and Padding
Fields must start at offsets divisible by their size. Compilers insert padding bytes to maintain this alignment. A DWORD (4 bytes) followed by a pointer (8 bytes on 64-bit) requires 4 bytes of padding so the pointer starts at offset 8 instead of 4.

## Solution 1: Working with RECT

The RECT structure represents a rectangle with four corners:
```c
typedef struct tagRECT {
    LONG left;    // 4 bytes at offset 0
    LONG top;     // 4 bytes at offset 4
    LONG right;   // 4 bytes at offset 8
    LONG bottom;  // 4 bytes at offset 12
} RECT;           // Total: 16 bytes
```

### Complete Working Example - GetWindowRect
```ahk
#Requires AutoHotkey v2.0

; Get window bounds in screen coordinates
GetWindowDimensions(hwnd) {
    ; RECT structure: 4 LONGs (4 bytes each) = 16 bytes total
    rect := Buffer(16, 0)

    result := DllCall("GetWindowRect", "Ptr", hwnd, "Ptr", rect)
    if (!result) {
        MsgBox "GetWindowRect failed. Error: " A_LastError
        return false
    }

    return {
        left: NumGet(rect, 0, "Int"),
        top: NumGet(rect, 4, "Int"),
        right: NumGet(rect, 8, "Int"),
        bottom: NumGet(rect, 12, "Int"),
        width: NumGet(rect, 8, "Int") - NumGet(rect, 0, "Int"),
        height: NumGet(rect, 12, "Int") - NumGet(rect, 4, "Int")
    }
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    dims := GetWindowDimensions(hwnd)
    MsgBox Format("Window bounds:`nLeft: {1}, Top: {2}`nRight: {3}, Bottom: {4}`nSize: {5}×{6}",
        dims.left, dims.top, dims.right, dims.bottom, dims.width, dims.height)
}
```

### Complete Working Example - GetClientRect
```ahk
; Get client area (inside borders/titlebar)
GetClientArea(hwnd) {
    rect := Buffer(16, 0)
    result := DllCall("GetClientRect", "Ptr", hwnd, "Ptr", rect)

    if (!result) {
        MsgBox "GetClientRect failed. Error: " A_LastError
        return false
    }

    ; Client rect left and top are always 0
    return {
        width: NumGet(rect, 8, "Int"),   ; right - left (left always 0)
        height: NumGet(rect, 12, "Int")  ; bottom - top (top always 0)
    }
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    client := GetClientArea(hwnd)
    MsgBox "Client area: " client.width "×" client.height " pixels"
}
```

## Solution 2: Working with POINT

The POINT structure represents a coordinate pair:
```c
typedef struct tagPOINT {
    LONG x;  // 4 bytes at offset 0
    LONG y;  // 4 bytes at offset 4
} POINT;     // Total: 8 bytes
```

### Complete Working Example - GetCursorPos
```ahk
; Get current mouse cursor position in screen coordinates
GetMousePosition() {
    pt := Buffer(8, 0)  ; 2 Ints = 8 bytes
    DllCall("GetCursorPos", "Ptr", pt)

    return {
        x: NumGet(pt, 0, "Int"),
        y: NumGet(pt, 4, "Int")
    }
}

; Test it
pos := GetMousePosition()
MsgBox "Mouse is at: " pos.x ", " pos.y

; Continuous tracking example
^Esc::ExitApp  ; Ctrl+Esc to exit

SetTimer TrackMouse, 100
TrackMouse() {
    pos := GetMousePosition()
    ToolTip "X: " pos.x " Y: " pos.y
}
```

### Complete Working Example - ScreenToClient
```ahk
; Convert screen coordinates to client coordinates
ScreenToClient(hwnd, screenX, screenY) {
    pt := Buffer(8, 0)
    NumPut("Int", screenX, pt, 0)
    NumPut("Int", screenY, pt, 4)

    result := DllCall("ScreenToClient", "Ptr", hwnd, "Ptr", pt)
    if (!result) {
        MsgBox "ScreenToClient failed. Error: " A_LastError
        return false
    }

    return {
        x: NumGet(pt, 0, "Int"),
        y: NumGet(pt, 4, "Int")
    }
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    screenPos := GetMousePosition()
    clientPos := ScreenToClient(hwnd, screenPos.x, screenPos.y)
    MsgBox Format("Screen: ({1}, {2})`nClient: ({3}, {4})",
        screenPos.x, screenPos.y, clientPos.x, clientPos.y)
}
```

## Solution 3: Working with SYSTEMTIME

The SYSTEMTIME structure contains all time components:
```c
typedef struct _SYSTEMTIME {
    WORD wYear;           // offset 0  (2 bytes)
    WORD wMonth;          // offset 2  (2 bytes)
    WORD wDayOfWeek;      // offset 4  (2 bytes)
    WORD wDay;            // offset 6  (2 bytes)
    WORD wHour;           // offset 8  (2 bytes)
    WORD wMinute;         // offset 10 (2 bytes)
    WORD wSecond;         // offset 12 (2 bytes)
    WORD wMilliseconds;   // offset 14 (2 bytes)
} SYSTEMTIME;             // Total: 16 bytes
```

**Important:** All fields are UShort (WORD), not Int. This structure has no padding because all fields are the same size.

### Complete Working Example - GetSystemTime
```ahk
; Get current UTC time
GetCurrentSystemTime() {
    st := Buffer(16, 0)  ; 8 WORDs = 16 bytes
    DllCall("GetSystemTime", "Ptr", st)

    return {
        year: NumGet(st, 0, "UShort"),
        month: NumGet(st, 2, "UShort"),
        dayOfWeek: NumGet(st, 4, "UShort"),
        day: NumGet(st, 6, "UShort"),
        hour: NumGet(st, 8, "UShort"),
        minute: NumGet(st, 10, "UShort"),
        second: NumGet(st, 12, "UShort"),
        milliseconds: NumGet(st, 14, "UShort")
    }
}

; Test it
time := GetCurrentSystemTime()
MsgBox Format("UTC Time: {:04d}-{:02d}-{:02d} {:02d}:{:02d}:{:02d}.{:03d}",
    time.year, time.month, time.day,
    time.hour, time.minute, time.second, time.milliseconds)
```

### Complete Working Example - GetLocalTime
```ahk
; Get current local time
GetCurrentLocalTime() {
    st := Buffer(16, 0)
    DllCall("GetLocalTime", "Ptr", st)

    return {
        year: NumGet(st, 0, "UShort"),
        month: NumGet(st, 2, "UShort"),
        dayOfWeek: NumGet(st, 4, "UShort"),
        day: NumGet(st, 6, "UShort"),
        hour: NumGet(st, 8, "UShort"),
        minute: NumGet(st, 10, "UShort"),
        second: NumGet(st, 12, "UShort"),
        milliseconds: NumGet(st, 14, "UShort")
    }
}

; Day names for display
DayName(dow) {
    static days := ["Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"]
    return days[dow + 1]
}

; Test it with formatted output
time := GetCurrentLocalTime()
MsgBox Format("Local Time:`n{1}, {:04d}-{:02d}-{:02d}`n{:02d}:{:02d}:{:02d}.{:03d}",
    DayName(time.dayOfWeek),
    time.year, time.month, time.day,
    time.hour, time.minute, time.second, time.milliseconds)
```

### Complete Working Example - SetSystemTime
```ahk
; Set a custom SYSTEMTIME (requires admin rights)
SetCustomSystemTime(year, month, day, hour, minute, second) {
    st := Buffer(16, 0)

    ; Write all fields as UShort
    NumPut("UShort", year, st, 0)
    NumPut("UShort", month, st, 2)
    NumPut("UShort", 0, st, 4)        ; dayOfWeek (ignored, auto-calculated)
    NumPut("UShort", day, st, 6)
    NumPut("UShort", hour, st, 8)
    NumPut("UShort", minute, st, 10)
    NumPut("UShort", second, st, 12)
    NumPut("UShort", 0, st, 14)       ; milliseconds

    result := DllCall("SetSystemTime", "Ptr", st, "Int")
    if (!result) {
        MsgBox "SetSystemTime failed. Error: " A_LastError
        return false
    }
    return true
}

; Example: Set to New Year 2025 (requires admin)
; if (!A_IsAdmin) {
;     Run '*RunAs "' A_ScriptFullPath '"'
;     ExitApp
; }
; SetCustomSystemTime(2025, 1, 1, 0, 0, 0)
```

## Solution 4: Working with MONITORINFO

The MONITORINFO structure is more complex because it contains a size field that must be initialized:
```c
typedef struct tagMONITORINFO {
    DWORD cbSize;        // offset 0,  4 bytes
    RECT  rcMonitor;     // offset 4,  16 bytes
    RECT  rcWork;        // offset 20, 16 bytes
    DWORD dwFlags;       // offset 36, 4 bytes
} MONITORINFO;           // Total: 40 bytes
```

**Critical requirement:** The `cbSize` field must be set to the structure's size before calling the API.

### Complete Working Example - GetMonitorInfo
```ahk
; Get monitor information for a window
GetMonitorInfo(hwnd) {
    ; First get the monitor handle from the window
    hMonitor := DllCall("MonitorFromWindow", "Ptr", hwnd,
                       "UInt", 0x2, "Ptr")  ; MONITOR_DEFAULTTONEAREST

    if (!hMonitor) {
        MsgBox "MonitorFromWindow failed. Error: " A_LastError
        return false
    }

    ; Allocate MONITORINFO struct (40 bytes)
    info := Buffer(40, 0)
    NumPut("UInt", 40, info, 0)  ; CRITICAL: Set cbSize field

    result := DllCall("GetMonitorInfo", "Ptr", hMonitor, "Ptr", info)
    if (!result) {
        MsgBox "GetMonitorInfo failed. Error: " A_LastError
        return false
    }

    return {
        ; Monitor bounds (rcMonitor at offset 4)
        monLeft: NumGet(info, 4, "Int"),
        monTop: NumGet(info, 8, "Int"),
        monRight: NumGet(info, 12, "Int"),
        monBottom: NumGet(info, 16, "Int"),

        ; Work area bounds (rcWork at offset 20)
        workLeft: NumGet(info, 20, "Int"),
        workTop: NumGet(info, 24, "Int"),
        workRight: NumGet(info, 28, "Int"),
        workBottom: NumGet(info, 32, "Int"),

        ; Flags (dwFlags at offset 36)
        isPrimary: NumGet(info, 36, "UInt") & 1
    }
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    mon := GetMonitorInfo(hwnd)
    MsgBox Format("Monitor Info:`n" .
        "Full: {1},{2} to {3},{4} ({5}×{6})`n" .
        "Work: {7},{8} to {9},{10} ({11}×{12})`n" .
        "Primary: {13}",
        mon.monLeft, mon.monTop, mon.monRight, mon.monBottom,
        mon.monRight - mon.monLeft, mon.monBottom - mon.monTop,
        mon.workLeft, mon.workTop, mon.workRight, mon.workBottom,
        mon.workRight - mon.workLeft, mon.workBottom - mon.workTop,
        mon.isPrimary ? "Yes" : "No")
}
```

### Complete Working Example - Get Work Area
```ahk
; Get work area (screen minus taskbar) for any window or primary monitor
GetWorkArea(hwnd := 0) {
    ; If no hwnd, use primary monitor
    if (!hwnd)
        hMonitor := DllCall("MonitorFromPoint", "Int64", 0, "UInt", 0x1, "Ptr")
    else
        hMonitor := DllCall("MonitorFromWindow", "Ptr", hwnd, "UInt", 0x2, "Ptr")

    info := Buffer(40, 0)
    NumPut("UInt", 40, info, 0)  ; cbSize

    if (!DllCall("GetMonitorInfo", "Ptr", hMonitor, "Ptr", info)) {
        MsgBox "GetMonitorInfo failed. Error: " A_LastError
        return false
    }

    ; rcWork starts at offset 20
    return {
        x: NumGet(info, 20, "Int"),
        y: NumGet(info, 24, "Int"),
        width: NumGet(info, 28, "Int") - NumGet(info, 20, "Int"),
        height: NumGet(info, 32, "Int") - NumGet(info, 24, "Int")
    }
}

; Test it - Center window in work area
hwnd := WinExist("A")
if (hwnd) {
    work := GetWorkArea(hwnd)
    WinGetPos(&x, &y, &w, &h, hwnd)

    newX := work.x + (work.width - w) // 2
    newY := work.y + (work.height - h) // 2

    WinMove(newX, newY,,, hwnd)
    MsgBox "Window centered in work area"
}
```

### Complete Working Example - Monitor from Point
```ahk
; Get which monitor contains a specific point
GetMonitorFromPoint(x, y) {
    ; Pack coordinates into Int64 (POINT passed by value)
    point := (y << 32) | (x & 0xFFFFFFFF)

    hMonitor := DllCall("MonitorFromPoint", "Int64", point,
                       "UInt", 0x2, "Ptr")  ; MONITOR_DEFAULTTONEAREST

    if (!hMonitor)
        return false

    info := Buffer(40, 0)
    NumPut("UInt", 40, info, 0)

    if (!DllCall("GetMonitorInfo", "Ptr", hMonitor, "Ptr", info))
        return false

    return {
        left: NumGet(info, 4, "Int"),
        top: NumGet(info, 8, "Int"),
        right: NumGet(info, 12, "Int"),
        bottom: NumGet(info, 16, "Int"),
        isPrimary: NumGet(info, 36, "UInt") & 1
    }
}

; Test it - Get monitor at mouse cursor
pos := GetMousePosition()
mon := GetMonitorFromPoint(pos.x, pos.y)
if (mon) {
    MsgBox Format("Monitor at cursor:`n{1},{2} to {3},{4}`nPrimary: {5}",
        mon.left, mon.top, mon.right, mon.bottom,
        mon.isPrimary ? "Yes" : "No")
}
```

## Solution 5: Working with FLASHWINFO (Complex Struct)

This structure demonstrates handling alignment issues on 32-bit vs 64-bit systems:
```c
typedef struct {
    UINT  cbSize;      // 4 bytes at offset 0
    HWND  hwnd;        // 4/8 bytes at offset 4/4
    DWORD dwFlags;     // 4 bytes at offset 8/12
    UINT  uCount;      // 4 bytes at offset 12/16
    DWORD dwTimeout;   // 4 bytes at offset 16/20
} FLASHWINFO;          // Total: 20 bytes (32-bit) or 24 bytes (64-bit)
```

### Complete Working Example - FlashWindowEx
```ahk
; Flash window to get user attention
FlashWindow(hwnd, flashCount := 5, flags := 0x0F) {
    ; Calculate size based on pointer size
    ; 32-bit: 4 + 4 + 4 + 4 + 4 = 20
    ; 64-bit: 4 + 8 + 4 + 4 + 4 = 24 (with padding after HWND)
    structSize := 20 + (A_PtrSize - 4)  ; Adjust for 64-bit HWND
    fwi := Buffer(structSize, 0)

    ; Fill structure carefully considering alignment
    offset := 0
    NumPut("UInt", structSize, fwi, offset)
    offset += 4
    NumPut("Ptr", hwnd, fwi, offset)
    offset += A_PtrSize
    NumPut("UInt", flags, fwi, offset)  ; FLASHW_ALL | FLASHW_TIMERNOFG
    offset += 4
    NumPut("UInt", flashCount, fwi, offset)
    offset += 4
    NumPut("UInt", 0, fwi, offset)     ; Default timeout

    result := DllCall("FlashWindowEx", "Ptr", fwi, "Int")
    if (!result) {
        MsgBox "FlashWindowEx failed. Error: " A_LastError
        return false
    }
    return true
}

; Flash window flag constants
FLASHW_STOP := 0       ; Stop flashing
FLASHW_CAPTION := 0x1  ; Flash caption
FLASHW_TRAY := 0x2     ; Flash taskbar button
FLASHW_ALL := 0x3      ; Flash both
FLASHW_TIMER := 0x4    ; Flash continuously
FLASHW_TIMERNOFG := 0xC ; Flash until window comes to foreground

; Test it
hwnd := WinExist("A")
if (hwnd) {
    FlashWindow(hwnd, 5, FLASHW_ALL)
    MsgBox "Window flashed 5 times"
}
```

## Troubleshooting

### Wrong Struct Size
**Symptom:** Function returns 0, A_LastError shows invalid parameter.
**Solution:** Verify total struct size. Count all fields and account for padding. On 64-bit, pointers add 8 bytes, not 4.

### Alignment Issues
**Symptom:** Values read back are incorrect or corrupted.
**Solution:** Fields must align to their size. Use A_PtrSize to calculate dynamic offsets. A UInt (4 bytes) after a Ptr (8 bytes) on 64-bit needs no padding, but a Ptr after a UInt needs 4 bytes padding.

### Size Field Not Set
**Symptom:** Function fails even though struct looks correct.
**Solution:** Many structs (MONITORINFO, FLASHWINFO, etc.) require cbSize as the first field. Always set it: `NumPut("UInt", structSize, buffer, 0)`.

### Wrong Type in NumGet/NumPut
**Symptom:** Values are wrong, negative numbers appear, or very large values.
**Solution:** Match types exactly. WORD is UShort (2 bytes), DWORD is UInt (4 bytes), LONG is Int (4 bytes). Using Int for UInt can make large values appear negative.

### Wrong Offsets
**Symptom:** Some fields read correctly, others are garbage.
**Solution:** Calculate offsets manually. Don't assume fields are back-to-back—padding may exist. Verify against MSDN documentation or C struct definitions.

### Forgot to Zero-Fill Buffer
**Symptom:** Intermittent failures or random values.
**Solution:** Always use `Buffer(size, 0)` with the second parameter. Uninitialized memory contains random data.

### Platform Differences
**Symptom:** Works on 32-bit, fails on 64-bit (or vice versa).
**Solution:** Use A_PtrSize for pointer fields. Test on both architectures. Struct size changes between platforms if it contains pointers.

## Cross-References

For theory on DllCall mechanics and type system: docs/manuals/ahk2/reference/dllcall-types.md
For debugging DllCall failures systematically: docs/manuals/ahk2/how-to/debug-dllcall-errors.md
For handling output parameters and buffers: docs/manuals/ahk2/how-to/dllcall-output-parameters.md
For reading MSDN documentation: docs/manuals/ahk2/explanation/windows-api-translation.md
