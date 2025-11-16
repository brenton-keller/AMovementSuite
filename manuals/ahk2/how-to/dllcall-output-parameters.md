# How-To: Handle DllCall Output Parameters

## Problem

Many Windows API functions write results to buffers or output parameters you provide, rather than just returning a simple value. You need to allocate memory correctly, use the right syntax for output parameters, and extract the results after the call completes.

## Quick Context

**Input vs Output Parameters:** Input parameters pass data TO the function (you provide the value). Output parameters receive data FROM the function (the function writes to your variable or buffer). Some parameters are both input and output.

**Buffer Management:** For output, you must allocate memory BEFORE calling the function. The function writes into your pre-allocated buffer. If you don't allocate enough space, the function may fail or cause memory corruption.

**Two Patterns:** Simple output parameters use the `Type*` notation with `&variable`. Buffer outputs use pre-allocated Buffer objects and require extracting data afterward with NumGet or StrGet.

## Key Concepts

### Type* Notation for Output Parameters

The asterisk (`*`) after a type means "pass the address of this variable." In AHK v2, you must use the VarRef operator `&`:

```ahk
processId := 0
threadId := DllCall("GetWindowThreadProcessId", "Ptr", hwnd, "UInt*", &processId, "UInt")
; Now processId contains the value written by the function
```

**Critical v2 change:** In v1 you wrote `"UInt*", processId`. In v2 you MUST write `"UInt*", &processId`.

### Pre-Allocated Buffers

For strings and structures, allocate a Buffer object:
```ahk
buffer := Buffer(size, 0)  ; Allocate and zero-fill
DllCall("SomeFunction", "Ptr", buffer, ...)
result := StrGet(buffer)   ; Extract the string
```

The second parameter to Buffer (0) zero-fills the memory, which is safer than uninitialized memory.

### StrGet for String Extraction

After a function writes a string to a buffer, use StrGet to convert bytes to an AHK string:
```ahk
title := StrGet(buffer, "UTF-16")  ; Windows uses UTF-16
```

Common encodings:
- `"UTF-16"` - Windows Unicode strings (wide char)
- `"CP0"` - ANSI using system default code page
- `"UTF-8"` - UTF-8 encoded strings

### Two-Call Pattern

Some functions require calling twice:
1. First call with NULL buffer to get required size
2. Second call with properly sized buffer to get data

```ahk
size := 0
DllCall("SomeFunction", "Ptr", 0, "Int*", &size)  ; Get size
buffer := Buffer(size, 0)                          ; Allocate
DllCall("SomeFunction", "Ptr", buffer, "Int*", &size)  ; Get data
```

## Solution 1: Numeric Output Parameters

These use `Type*` notation and the `&` operator.

### Complete Working Example - GetWindowThreadProcessId
```ahk
#Requires AutoHotkey v2.0

; Get both thread ID and process ID from a window
GetWindowProcess(hwnd) {
    processId := 0
    threadId := DllCall("GetWindowThreadProcessId", "Ptr", hwnd,
                       "UInt*", &processId, "UInt")

    if (!threadId) {
        MsgBox "GetWindowThreadProcessId failed. Error: " A_LastError
        return false
    }

    return {threadId: threadId, processId: processId}
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    info := GetWindowProcess(hwnd)
    MsgBox Format("Window Info:`nThread ID: {1}`nProcess ID: {2}",
        info.threadId, info.processId)
}
```

### Complete Working Example - GetCursorPos with Output
```ahk
; Alternative approach using output parameter pattern
GetMousePositionAlt() {
    x := 0
    y := 0

    ; Note: GetCursorPos actually takes a struct pointer,
    ; but here's how you'd use output params if it worked that way
    pt := Buffer(8, 0)
    result := DllCall("GetCursorPos", "Ptr", pt)

    if (!result) {
        MsgBox "GetCursorPos failed. Error: " A_LastError
        return false
    }

    return {
        x: NumGet(pt, 0, "Int"),
        y: NumGet(pt, 4, "Int")
    }
}

; Test it
pos := GetMousePositionAlt()
MsgBox "Mouse at: " pos.x ", " pos.y
```

### Complete Working Example - GetExitCodeProcess
```ahk
; Get exit code of a process
GetProcessExitCode(hProcess) {
    exitCode := 0
    result := DllCall("GetExitCodeProcess", "Ptr", hProcess, "UInt*", &exitCode)

    if (!result) {
        MsgBox "GetExitCodeProcess failed. Error: " A_LastError
        return false
    }

    ; STILL_ACTIVE = 259 means process is still running
    if (exitCode = 259)
        return {active: true, exitCode: exitCode}
    else
        return {active: false, exitCode: exitCode}
}

; Example: Monitor a process
; hProcess := DllCall("OpenProcess", "UInt", 0x1000, "Int", 0, "UInt", processId, "Ptr")
; status := GetProcessExitCode(hProcess)
; DllCall("CloseHandle", "Ptr", hProcess)
```

### Complete Working Example - GetVersionEx
```ahk
; Get Windows version (legacy API, but demonstrates output struct)
GetWindowsVersion() {
    ; OSVERSIONINFO structure (32-bit: 148 bytes, 64-bit: 148 bytes)
    osvi := Buffer(148, 0)
    NumPut("UInt", 148, osvi, 0)  ; dwOSVersionInfoSize

    result := DllCall("GetVersionEx", "Ptr", osvi)
    if (!result) {
        MsgBox "GetVersionEx failed. Error: " A_LastError
        return false
    }

    return {
        majorVersion: NumGet(osvi, 4, "UInt"),
        minorVersion: NumGet(osvi, 8, "UInt"),
        buildNumber: NumGet(osvi, 12, "UInt"),
        platformId: NumGet(osvi, 16, "UInt")
    }
}

; Test it
version := GetWindowsVersion()
if (version) {
    MsgBox Format("Windows Version: {1}.{2} Build {3}",
        version.majorVersion, version.minorVersion, version.buildNumber)
}
```

## Solution 2: String Output Parameters

String outputs require pre-allocated buffers and character count parameters.

### Complete Working Example - GetWindowText
```ahk
; Get window title with proper buffer handling
GetWindowTitle(hwnd) {
    ; Get required length first
    length := DllCall("GetWindowTextLength", "Ptr", hwnd, "Int") + 1

    if (length <= 1) {
        ; No title or error
        return ""
    }

    ; Allocate buffer (characters × 2 for UTF-16)
    buffer := Buffer(length * 2, 0)

    ; Get the title
    chars := DllCall("GetWindowText", "Ptr", hwnd,
                    "Ptr", buffer, "Int", length, "Int")

    if (chars = 0 && A_LastError != 0) {
        MsgBox "GetWindowText failed. Error: " A_LastError
        return ""
    }

    ; Extract string
    return StrGet(buffer, "UTF-16")
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    title := GetWindowTitle(hwnd)
    MsgBox "Window title: " title
}
```

**Key points:**
- GetWindowTextLength returns character count, not byte count
- Buffer size = characters × 2 (UTF-16 uses 2 bytes per character)
- The length parameter to GetWindowText is in characters
- StrGet extracts the string from the buffer

### Complete Working Example - GetClassName
```ahk
; Get window class name
GetWindowClassName(hwnd) {
    MAX_CLASS_NAME := 256
    buffer := Buffer(MAX_CLASS_NAME * 2, 0)  ; *2 for UTF-16

    length := DllCall("GetClassName", "Ptr", hwnd,
                     "Ptr", buffer, "Int", MAX_CLASS_NAME, "Int")

    if (!length) {
        MsgBox "GetClassName failed. Error: " A_LastError
        return ""
    }

    return StrGet(buffer, "UTF-16")
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    className := GetWindowClassName(hwnd)
    MsgBox "Window class: " className
}
```

### Complete Working Example - GetModuleFileName
```ahk
; Get full path to executable for a process
GetProcessPath(hProcess := 0) {
    MAX_PATH := 260
    buffer := Buffer(MAX_PATH * 2, 0)

    ; If no process handle, get current process
    if (!hProcess)
        hProcess := DllCall("GetCurrentProcess", "Ptr")

    length := DllCall("GetModuleFileNameEx", "Ptr", hProcess,
                     "Ptr", 0,  ; NULL for main executable
                     "Ptr", buffer, "UInt", MAX_PATH, "UInt")

    if (!length) {
        MsgBox "GetModuleFileNameEx failed. Error: " A_LastError
        return ""
    }

    return StrGet(buffer, "UTF-16")
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    info := GetWindowProcess(hwnd)
    hProcess := DllCall("OpenProcess", "UInt", 0x1000, "Int", 0,
                       "UInt", info.processId, "Ptr")

    if (hProcess) {
        path := GetProcessPath(hProcess)
        DllCall("CloseHandle", "Ptr", hProcess)
        MsgBox "Executable: " path
    }
}
```

### Complete Working Example - GetEnvironmentVariable
```ahk
; Get environment variable value
GetEnvVar(varName) {
    ; First call to get required size
    size := DllCall("GetEnvironmentVariable", "Str", varName, "Ptr", 0, "UInt", 0, "UInt")

    if (!size) {
        ; Variable doesn't exist
        return ""
    }

    ; Allocate buffer (size is in characters, includes null terminator)
    buffer := Buffer(size * 2, 0)

    ; Second call to get value
    result := DllCall("GetEnvironmentVariable", "Str", varName,
                     "Ptr", buffer, "UInt", size, "UInt")

    if (!result) {
        MsgBox "GetEnvironmentVariable failed. Error: " A_LastError
        return ""
    }

    return StrGet(buffer, "UTF-16")
}

; Test it
userProfile := GetEnvVar("USERPROFILE")
MsgBox "User Profile: " userProfile

temp := GetEnvVar("TEMP")
MsgBox "Temp folder: " temp
```

## Solution 3: Two-Call Pattern

Many functions use this pattern: call once to get size, call again to get data.

### Complete Working Example - GetWindowModuleFileName
```ahk
; Get executable path from window handle
GetWindowExePath(hwnd) {
    ; First call: get required buffer size
    length := DllCall("GetWindowModuleFileName", "Ptr", hwnd,
                     "Ptr", 0, "UInt", 0, "UInt")

    if (!length) {
        MsgBox "GetWindowModuleFileName failed. Error: " A_LastError
        return ""
    }

    ; Second call: get actual path
    buffer := Buffer(length * 2, 0)
    result := DllCall("GetWindowModuleFileName", "Ptr", hwnd,
                     "Ptr", buffer, "UInt", length, "UInt")

    if (!result) {
        MsgBox "GetWindowModuleFileName failed on second call. Error: " A_LastError
        return ""
    }

    return StrGet(buffer, "UTF-16")
}

; Test it
hwnd := WinExist("A")
if (hwnd) {
    path := GetWindowExePath(hwnd)
    MsgBox "Executable: " path
}
```

### Complete Working Example - QueryDosDevice
```ahk
; Get device mapping (e.g., C: -> \Device\HarddiskVolume1)
QueryDosDevice(deviceName) {
    ; First call to get required size
    size := DllCall("QueryDosDevice", "Str", deviceName, "Ptr", 0, "UInt", 0, "UInt")

    if (!size) {
        MsgBox "QueryDosDevice failed. Error: " A_LastError
        return ""
    }

    ; Second call to get actual mapping
    buffer := Buffer(size * 2, 0)
    result := DllCall("QueryDosDevice", "Str", deviceName,
                     "Ptr", buffer, "UInt", size, "UInt")

    if (!result) {
        MsgBox "QueryDosDevice failed on second call. Error: " A_LastError
        return ""
    }

    return StrGet(buffer, "UTF-16")
}

; Test it
mapping := QueryDosDevice("C:")
MsgBox "C: maps to: " mapping
```

### Complete Working Example - GetLogicalDriveStrings
```ahk
; Get all drive letters
GetLogicalDrives() {
    ; Get required buffer size
    size := DllCall("GetLogicalDriveStrings", "UInt", 0, "Ptr", 0, "UInt")

    if (!size) {
        MsgBox "GetLogicalDriveStrings failed. Error: " A_LastError
        return []
    }

    ; Get drive strings
    buffer := Buffer(size * 2, 0)
    result := DllCall("GetLogicalDriveStrings", "UInt", size, "Ptr", buffer, "UInt")

    if (!result) {
        MsgBox "GetLogicalDriveStrings failed. Error: " A_LastError
        return []
    }

    ; Parse null-separated list
    drives := []
    offset := 0
    loop {
        drive := StrGet(buffer.Ptr + offset, "UTF-16")
        if (drive = "")
            break
        drives.Push(drive)
        offset += (StrLen(drive) + 1) * 2  ; +1 for null, *2 for UTF-16
    }

    return drives
}

; Test it
drives := GetLogicalDrives()
driveList := ""
for drive in drives
    driveList .= drive "`n"
MsgBox "Logical drives:`n" driveList
```

## Solution 4: File Operations with Buffers

Reading and writing files requires output buffers.

### Complete Working Example - ReadFile
```ahk
; Read file contents using Windows API
ReadFileContents(filename, maxSize := 65536) {
    ; Open file
    hFile := DllCall("CreateFile", "Str", filename,
        "UInt", 0x80000000,    ; GENERIC_READ
        "UInt", 0x1,           ; FILE_SHARE_READ
        "Ptr", 0,              ; No security attributes
        "UInt", 3,             ; OPEN_EXISTING
        "UInt", 0,             ; Normal attributes
        "Ptr", 0,              ; No template
        "Ptr")

    if (!hFile || hFile == -1) {
        MsgBox "CreateFile failed. Error: " A_LastError
        return ""
    }

    ; Allocate buffer
    buffer := Buffer(maxSize, 0)
    bytesRead := 0

    ; Read file
    result := DllCall("ReadFile",
        "Ptr", hFile,
        "Ptr", buffer,
        "UInt", maxSize,
        "UInt*", &bytesRead,
        "Ptr", 0,
        "Int")

    DllCall("CloseHandle", "Ptr", hFile)

    if (!result) {
        MsgBox "ReadFile failed. Error: " A_LastError
        return ""
    }

    ; Extract string (assuming UTF-8)
    return StrGet(buffer, bytesRead, "UTF-8")
}

; Test it
; content := ReadFileContents("C:\test.txt")
; MsgBox content
```

### Complete Working Example - WriteFile
```ahk
; Write string to file using Windows API
WriteFileContents(filename, content) {
    ; Create/overwrite file
    hFile := DllCall("CreateFile", "Str", filename,
        "UInt", 0x40000000,    ; GENERIC_WRITE
        "UInt", 0,             ; No sharing
        "Ptr", 0,              ; No security attributes
        "UInt", 2,             ; CREATE_ALWAYS
        "UInt", 0x80,          ; FILE_ATTRIBUTE_NORMAL
        "Ptr", 0,              ; No template
        "Ptr")

    if (!hFile || hFile == -1) {
        MsgBox "CreateFile failed. Error: " A_LastError
        return false
    }

    ; Convert string to UTF-8 bytes
    size := StrPut(content, "UTF-8") - 1  ; -1 to exclude null terminator
    buffer := Buffer(size, 0)
    StrPut(content, buffer, "UTF-8")

    ; Write to file
    bytesWritten := 0
    result := DllCall("WriteFile",
        "Ptr", hFile,
        "Ptr", buffer,
        "UInt", size,
        "UInt*", &bytesWritten,
        "Ptr", 0,
        "Int")

    DllCall("CloseHandle", "Ptr", hFile)

    if (!result) {
        MsgBox "WriteFile failed. Error: " A_LastError
        return false
    }

    return bytesWritten
}

; Test it
; bytesWritten := WriteFileContents("C:\test.txt", "Hello, World!")
; MsgBox "Wrote " bytesWritten " bytes"
```

### Complete Working Example - GetFileSize
```ahk
; Get file size using Windows API
GetFileSize(filename) {
    hFile := DllCall("CreateFile", "Str", filename,
        "UInt", 0x80000000,    ; GENERIC_READ
        "UInt", 0x1,           ; FILE_SHARE_READ
        "Ptr", 0,
        "UInt", 3,             ; OPEN_EXISTING
        "UInt", 0,
        "Ptr", 0,
        "Ptr")

    if (!hFile || hFile == -1) {
        MsgBox "CreateFile failed. Error: " A_LastError
        return false
    }

    sizeHigh := 0
    sizeLow := DllCall("GetFileSize", "Ptr", hFile, "UInt*", &sizeHigh, "UInt")

    DllCall("CloseHandle", "Ptr", hFile)

    if (sizeLow == 0xFFFFFFFF && A_LastError != 0) {
        MsgBox "GetFileSize failed. Error: " A_LastError
        return false
    }

    ; Combine high and low parts into 64-bit size
    return (sizeHigh << 32) | sizeLow
}

; Test it
; size := GetFileSize("C:\Windows\notepad.exe")
; MsgBox "File size: " size " bytes"
```

## Troubleshooting

### Missing VarRef & in v2
**Symptom:** Output parameter doesn't get updated, variable stays 0.
**Solution:** In AHK v2, you MUST use `&` before the variable name: `"UInt*", &processId` not `"UInt*", processId`.

### Buffer Too Small
**Symptom:** Function returns 0, A_LastError shows ERROR_INSUFFICIENT_BUFFER (122) or ERROR_MORE_DATA (234).
**Solution:** Increase buffer size or use two-call pattern to get required size first.

### Wrong Encoding for StrGet
**Symptom:** String contains garbage characters or question marks.
**Solution:** Windows uses UTF-16 for most APIs. Use `StrGet(buffer, "UTF-16")`. For ANSI functions, use `"CP0"`.

### Character Count vs Byte Count
**Symptom:** String is truncated or buffer is too small.
**Solution:** String functions take CHARACTER counts, not bytes. UTF-16 needs 2 bytes per character. Allocate `chars * 2` bytes.

### Forgot to Allocate Buffer
**Symptom:** Crash or access violation.
**Solution:** ALWAYS allocate buffer before calling: `buffer := Buffer(size, 0)`. Never pass uninitialized variables.

### Forgot Null Terminator Space
**Symptom:** String is truncated by one character.
**Solution:** Add 1 to the length: `length := DllCall("GetWindowTextLength", ...) + 1`.

### Wrong Type for Output Parameter
**Symptom:** Corrupted value or crash.
**Solution:** Match the type exactly. If MSDN says `LPDWORD` (pointer to DWORD), use `"UInt*", &var` not `"Ptr*", &var`.

### Didn't Check Return Value
**Symptom:** Extract garbage from buffer even though call failed.
**Solution:** Always check the function's return value before using output data.

## Cross-References

For theory on type system and type mapping: docs/manuals/ahk2/reference/dllcall-types.md
For working with structs and NumGet/NumPut: docs/manuals/ahk2/how-to/work-with-structs.md
For debugging DllCall failures: docs/manuals/ahk2/how-to/debug-dllcall-errors.md
For understanding memory layout: docs/manuals/ahk2/explanation/dllcall-memory-model.md
