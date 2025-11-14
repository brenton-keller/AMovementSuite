# How to Translate MSDN Documentation to DllCall

## Problem
You found a Windows API function on MSDN that does exactly what you need, but you don't know how to convert the C function signature into a working AutoHotkey DllCall.

## Quick Context
MSDN (Microsoft Developer Network) documents all Windows API functions with standardized C signatures. Each parameter has a specific type (HWND, DWORD, LPSTR, etc.) and an annotation ([in], [out], [optional]) that tells you how to use it. Converting these to AutoHotkey requires systematic type translation and understanding what the annotations mean.

## Key Concepts

**Parameter Annotations**
- `[in]` - You provide the value to the function
- `[out]` - Function writes the value (you pass a variable's address)
- `[in, out]` - Function reads and modifies (pass address with &var)
- `[optional]` - You can pass NULL/0 if you don't need it

**Type Categories**
- Handle types (HWND, HANDLE, HDC) → Always use `Ptr`
- Integer types (DWORD, UINT, LONG) → Use specific sized integers
- String types (LPSTR, LPWSTR) → Use string types or Ptr for output
- Pointer types (LPVOID, void*) → Always use `Ptr`

**A/W Variants**
Many string functions come in two versions: FunctionA (ANSI) and FunctionW (Unicode/Wide). Modern Windows uses Unicode internally, so prefer W variants or use the generic name (CreateFile auto-selects CreateFileW in v2).

## Complete Type Translation Table

| Windows Type | What It Is | AHK Type | Size | Notes |
|--------------|------------|----------|------|-------|
| **BOOL, BOOLEAN** | Integer 0/1 | Int | 4 bytes | Check for nonzero, not exactly 1 |
| **BYTE** | Unsigned char | UChar | 1 byte | Range 0-255 |
| **WORD** | Unsigned short | UShort | 2 bytes | 16-bit unsigned |
| **DWORD, ULONG** | Unsigned long | UInt | 4 bytes | 32-bit unsigned, very common |
| **LONG** | Long integer | Int | 4 bytes | 32-bit signed |
| **LONGLONG** | 64-bit integer | Int64 | 8 bytes | Large numbers |
| **UINT_PTR, SIZE_T** | Pointer-sized unsigned | UPtr | 4/8 bytes | Use for sizes that vary by platform |
| **INT_PTR, SSIZE_T** | Pointer-sized signed | Ptr | 4/8 bytes | Rare for actual integers |
| **HANDLE, HWND, HDC, HBITMAP** | Handle (void*) | **Ptr** | 4/8 bytes | **CRITICAL: Always Ptr, never Int** |
| **LPVOID, PVOID** | Generic pointer | Ptr | 4/8 bytes | Pointer to anything |
| **LPSTR, LPCSTR** | ANSI string pointer | AStr or Ptr | 4/8 bytes | Use AStr for input, Ptr+Buffer for output |
| **LPWSTR, LPCWSTR** | Unicode string pointer | WStr or Ptr | 4/8 bytes | Use WStr for input, Ptr+Buffer for output |
| **LPTSTR, LPCTSTR** | Auto string pointer | Str or Ptr | 4/8 bytes | Auto-selects based on UNICODE define |

**Type Prefix Quick Reference**
- **H** prefix (HWND, HANDLE) = Handle → Use `Ptr`
- **LP** or **P** prefix = Long Pointer → Use `Ptr`
- **C** in name (LPCSTR) = Const → Still use appropriate type
- **W** suffix = Wide/Unicode → Preferred on modern Windows
- **A** suffix = ANSI → Legacy, creates conversion overhead

## Translation Process Step-by-Step

### Step 1: Find the MSDN Function Page
Search for the function name + "msdn" (e.g., "SetWindowPos msdn"). The page shows:
- Function signature in C
- Parameter descriptions with annotations
- Return value explanation
- Links to related constants and structures

### Step 2: Extract the Function Signature
Example from SetWindowPos MSDN page:
```c
BOOL SetWindowPos(
    [in]           HWND hWnd,
    [in, optional] HWND hWndInsertAfter,
    [in]           int  X,
    [in]           int  Y,
    [in]           int  cx,
    [in]           int  cy,
    [in]           UINT uFlags
);
```

### Step 3: Translate Each Parameter
Go line-by-line through the signature:

**Parameter 1: `[in] HWND hWnd`**
- Type: HWND (handle to window)
- Direction: [in] (you provide it)
- Translation: `"Ptr", hwndVariable`

**Parameter 2: `[in, optional] HWND hWndInsertAfter`**
- Type: HWND
- Direction: [in, optional] (can pass 0)
- Translation: `"Ptr", 0` or `"Ptr", someHandle`

**Parameter 3-6: `[in] int X/Y/cx/cy`**
- Type: int (32-bit signed integer)
- Direction: [in]
- Translation: `"Int", xValue`, `"Int", yValue`, etc.

**Parameter 7: `[in] UINT uFlags`**
- Type: UINT (32-bit unsigned integer)
- Direction: [in]
- Translation: `"UInt", flagValue`

### Step 4: Identify Return Type
```c
BOOL SetWindowPos(...)
```
Return type is BOOL (32-bit int). In DllCall, specify return type as final parameter:
```ahk
result := DllCall("SetWindowPos", ..., "Int")
```

For pointer returns (HANDLE, HWND), use `"Ptr"`:
```ahk
handle := DllCall("LoadLibrary", "Str", "user32.dll", "Ptr")
```

### Step 5: Assemble Complete DllCall
```ahk
result := DllCall("SetWindowPos",
    "Ptr", hWnd,           ; HWND hWnd
    "Ptr", hWndInsertAfter, ; HWND hWndInsertAfter
    "Int", X,              ; int X
    "Int", Y,              ; int Y
    "Int", cx,             ; int cx (width)
    "Int", cy,             ; int cy (height)
    "UInt", uFlags,        ; UINT uFlags
    "Int")                 ; Return type: BOOL
```

## Complete Working Example 1: SetWindowPos

**MSDN Signature:**
```c
BOOL SetWindowPos(
    [in] HWND hWnd,
    [in, optional] HWND hWndInsertAfter,
    [in] int X,
    [in] int Y,
    [in] int cx,
    [in] int cy,
    [in] UINT uFlags
);
```

**Complete AHK Implementation:**
```ahk
; Make window always-on-top
MakeTopmost(hwnd) {
    ; HWND_TOPMOST = -1
    ; SWP_NOMOVE | SWP_NOSIZE = 0x0003
    result := DllCall("SetWindowPos",
        "Ptr", hwnd,        ; hWnd
        "Ptr", -1,          ; hWndInsertAfter (HWND_TOPMOST)
        "Int", 0,           ; X (ignored due to SWP_NOMOVE)
        "Int", 0,           ; Y (ignored due to SWP_NOMOVE)
        "Int", 0,           ; cx (ignored due to SWP_NOSIZE)
        "Int", 0,           ; cy (ignored due to SWP_NOSIZE)
        "UInt", 0x0003,     ; uFlags (SWP_NOMOVE | SWP_NOSIZE)
        "Int")              ; Return type

    if (!result) {
        MsgBox "SetWindowPos failed. Error: " A_LastError
        return false
    }
    return true
}

; Usage
hwnd := WinExist("A")
MakeTopmost(hwnd)
```

## Complete Working Example 2: CreateFile

**MSDN Signature:**
```c
HANDLE CreateFileW(
    [in]           LPCWSTR               lpFileName,
    [in]           DWORD                 dwDesiredAccess,
    [in]           DWORD                 dwShareMode,
    [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes,
    [in]           DWORD                 dwCreationDisposition,
    [in]           DWORD                 dwFlagsAndAttributes,
    [in, optional] HANDLE                hTemplateFile
);
```

**Translation Breakdown:**
- `LPCWSTR lpFileName` → `"WStr", filename` (Unicode string input)
- `DWORD dwDesiredAccess` → `"UInt", accessFlags` (32-bit unsigned)
- `DWORD dwShareMode` → `"UInt", shareFlags`
- `LPSECURITY_ATTRIBUTES` → `"Ptr", 0` (NULL for default security)
- `DWORD dwCreationDisposition` → `"UInt", createFlag`
- `DWORD dwFlagsAndAttributes` → `"UInt", attributes`
- `HANDLE hTemplateFile` → `"Ptr", 0` (optional, pass NULL)
- Return type `HANDLE` → `"Ptr"` (CRITICAL!)

**Complete AHK Implementation:**
```ahk
OpenFileForReading(filename) {
    ; Constants from MSDN
    GENERIC_READ := 0x80000000
    FILE_SHARE_READ := 0x1
    FILE_SHARE_WRITE := 0x2
    OPEN_EXISTING := 3
    FILE_ATTRIBUTE_NORMAL := 0x80
    INVALID_HANDLE_VALUE := -1

    hFile := DllCall("CreateFileW",
        "WStr", filename,                       ; lpFileName
        "UInt", GENERIC_READ,                   ; dwDesiredAccess
        "UInt", FILE_SHARE_READ | FILE_SHARE_WRITE, ; dwShareMode
        "Ptr", 0,                               ; lpSecurityAttributes (NULL)
        "UInt", OPEN_EXISTING,                  ; dwCreationDisposition
        "UInt", FILE_ATTRIBUTE_NORMAL,          ; dwFlagsAndAttributes
        "Ptr", 0,                               ; hTemplateFile (NULL)
        "Ptr")                                  ; Return type: HANDLE

    if (!hFile || hFile == INVALID_HANDLE_VALUE) {
        MsgBox "CreateFile failed for: " filename "`nError: " A_LastError
        return 0
    }

    return hFile
}

; Usage - remember to CloseHandle!
hFile := OpenFileForReading("C:\test.txt")
if (hFile) {
    ; Do file operations...
    DllCall("CloseHandle", "Ptr", hFile)
}
```

## Complete Working Example 3: GetMonitorInfo

**MSDN Signature:**
```c
BOOL GetMonitorInfoW(
    [in]  HMONITOR      hMonitor,
    [out] LPMONITORINFO lpmi
);
```

**Structure Definition:**
```c
typedef struct tagMONITORINFO {
    DWORD cbSize;      // Must be set to sizeof(MONITORINFO)
    RECT  rcMonitor;   // Full monitor rectangle
    RECT  rcWork;      // Work area (excludes taskbar)
    DWORD dwFlags;     // MONITORINFOF_PRIMARY if primary monitor
} MONITORINFO;
```

**Translation with Structure:**
- `HMONITOR hMonitor` → `"Ptr", monitorHandle`
- `LPMONITORINFO lpmi` → `"Ptr", bufferAddress` ([out] means we provide buffer)
- Structure size: DWORD (4) + RECT (16) + RECT (16) + DWORD (4) = 40 bytes
- First field `cbSize` must be set to 40 before calling

**Complete AHK Implementation:**
```ahk
GetMonitorWorkArea(hwnd) {
    ; First get monitor handle from window
    MONITOR_DEFAULTTONEAREST := 0x2
    hMonitor := DllCall("MonitorFromWindow",
        "Ptr", hwnd,
        "UInt", MONITOR_DEFAULTTONEAREST,
        "Ptr")

    if (!hMonitor) {
        MsgBox "MonitorFromWindow failed"
        return false
    }

    ; Allocate MONITORINFO structure (40 bytes)
    ; Structure layout:
    ; Offset 0:  cbSize (DWORD, 4 bytes)
    ; Offset 4:  rcMonitor (RECT, 16 bytes)
    ; Offset 20: rcWork (RECT, 16 bytes)
    ; Offset 36: dwFlags (DWORD, 4 bytes)

    info := Buffer(40, 0)
    NumPut("UInt", 40, info, 0)  ; CRITICAL: Set cbSize field

    result := DllCall("GetMonitorInfoW",
        "Ptr", hMonitor,           ; hMonitor
        "Ptr", info,               ; lpmi (pointer to our buffer)
        "Int")                     ; Return type: BOOL

    if (!result) {
        MsgBox "GetMonitorInfo failed. Error: " A_LastError
        return false
    }

    ; Extract work area rectangle (offset 20-35)
    return {
        x: NumGet(info, 20, "Int"),         ; rcWork.left
        y: NumGet(info, 24, "Int"),         ; rcWork.top
        width: NumGet(info, 28, "Int") - NumGet(info, 20, "Int"),  ; right - left
        height: NumGet(info, 32, "Int") - NumGet(info, 24, "Int"), ; bottom - top
        isPrimary: NumGet(info, 36, "UInt") & 1
    }
}

; Usage
hwnd := WinExist("A")
work := GetMonitorWorkArea(hwnd)
MsgBox "Work area: " work.width "x" work.height " at (" work.x ", " work.y ")"
```

## Finding Flag Values and Constants

### Method 1: MSDN Parameter Sections
Most MSDN pages list common flag values in the parameter description:
```
uFlags [in]
  Type: UINT

  The window sizing and positioning flags. Can be one or more of:
  - SWP_NOSIZE (0x0001): Retains current size
  - SWP_NOMOVE (0x0002): Retains current position
  ...
```

### Method 2: Search Constant Name
Google the constant name (e.g., "GENERIC_READ value") to find its hex value. Many results show:
```
GENERIC_READ = 0x80000000
```

### Method 3: Windows SDK Headers
Constants are defined in header files:
- `winuser.h` - Window management (SetWindowPos, ShowWindow)
- `winbase.h` - File operations (CreateFile)
- `wingdi.h` - Graphics (DC, fonts, drawing)

Download Windows SDK or browse headers online at Microsoft's GitHub.

### Method 4: PInvoke.net
Website [pinvoke.net](http://pinvoke.net) has searchable Windows API documentation with all constant values. Search for "SetWindowPos" to see complete flag list with values.

### Method 5: Use MsgBox to Test
If you find code that works but don't know the value:
```ahk
SWP_NOMOVE := 0x0002
SWP_NOSIZE := 0x0001
combined := SWP_NOMOVE | SWP_NOSIZE
MsgBox "Combined value: 0x" Format("{:X}", combined)  ; Shows: 0x3
```

## Troubleshooting Common Translation Issues

### Issue 1: Function Not Found (ErrorLevel -4)
**Symptom:** DllCall returns -4 or error message "function not found"

**Solutions:**
1. Check capitalization - exports are case-sensitive: `SetWindowPos` not `setwindowpos`
2. Try explicit W or A suffix: `"CreateFileW"` instead of `"CreateFile"`
3. Verify DLL name - user32.dll for window functions, kernel32.dll for files
4. Use Dependency Walker to see exact export name

### Issue 2: Truncated Handles on 64-bit
**Symptom:** Code works on 32-bit but fails on 64-bit AHK

**Solution:**
Change all handle types from `"Int"` to `"Ptr"`:
```ahk
; WRONG - works 32-bit only
hwnd := DllCall("FindWindow", "Str", "Notepad", "Str", "", "Int")

; CORRECT - works both 32 and 64-bit
hwnd := DllCall("FindWindow", "Str", "Notepad", "Str", "", "Ptr")
```

### Issue 3: String Output Shows Buffer Object
**Symptom:** Variable contains Buffer object instead of string

**Solution:**
Use StrGet to extract string from buffer:
```ahk
; WRONG - buffer contains binary data
buffer := Buffer(260 * 2, 0)
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 260)
MsgBox buffer  ; Shows [object Buffer]

; CORRECT - extract string
buffer := Buffer(260 * 2, 0)
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 260)
title := StrGet(buffer, "UTF-16")
MsgBox title  ; Shows actual window title
```

### Issue 4: Struct Documentation Missing Offsets
**Symptom:** MSDN shows structure members but you need byte offsets for NumGet

**Solution:**
Calculate manually based on type sizes:
```c
typedef struct {
    DWORD field1;    // Offset 0, size 4
    WORD  field2;    // Offset 4, size 2
    WORD  field3;    // Offset 6, size 2
    HWND  field4;    // Offset 8, size 4 or 8 (platform-dependent)
} MYSTRUCT;
```

Watch for alignment: fields must start at offsets divisible by their size (8-byte pointers must start at offset 0, 8, 16, etc.).

### Issue 5: Wrong Parameter Count
**Symptom:** ErrorLevel shows "A-16" or similar

**Solution:**
Count parameters carefully - easy to miss one:
```ahk
; WRONG - only 6 parameters, needs 7
DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
    "Int", 0, "Int", 0, "Int", 100, "Int", 100)

; CORRECT - all 7 parameters
DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
    "Int", 0, "Int", 0, "Int", 100, "Int", 100, "UInt", 0)
```

## Decision Matrix: When to Specify Return Type

| Function Returns | Specify Return Type | Example |
|------------------|---------------------|---------|
| BOOL, int, LONG | Optional (default Int) | `DllCall("SetWindowPos", ...)` |
| HANDLE, HWND, HDC, pointer | **REQUIRED: "Ptr"** | `DllCall("LoadLibrary", ..., "Ptr")` |
| DWORD, UINT | "UInt" | `DllCall("GetTickCount", "UInt")` |
| Int64, LONGLONG | "Int64" | `DllCall("GetFileSize64", ..., "Int64")` |
| void (no return) | Omit or "" | `DllCall("Sleep", "UInt", 1000)` |

**Critical Rule:** Always specify `"Ptr"` for functions returning handles or pointers. Omitting it truncates 64-bit values to 32 bits.

## Quick Reference Template

Use this template when translating new MSDN functions:

```ahk
; 1. Write MSDN signature as comment
; RETURNTYPE FunctionName(
;     [in] TYPE param1,
;     [out] TYPE param2,
;     ...
; );

; 2. Translate each parameter
result := DllCall("FunctionName",
    "AHKType", param1Value,    ; [in] TYPE param1
    "AHKType", param2Address,  ; [out] TYPE param2 - use &var
    "ReturnType")              ; RETURNTYPE

; 3. Check for errors
if (!result) {
    MsgBox "FunctionName failed: " A_LastError
    return false
}

; 4. Extract output if needed
if (isBufferOutput)
    output := NumGet(bufferVar, offset, "Type")
else if (isStringOutput)
    output := StrGet(bufferVar, "UTF-16")

return result
```

## Next Steps

Now that you can translate MSDN documentation:
1. Practice with simple functions (MessageBox, Beep, GetTickCount)
2. Try functions with structures (GetWindowRect, GetSystemTime)
3. Attempt file operations (CreateFile, ReadFile, WriteFile)
4. Explore complex APIs (RegisterClassEx, CreateWindow)

Remember: Every Windows API function follows the same pattern. Once you master type translation, you can implement any documented API function in AutoHotkey.
