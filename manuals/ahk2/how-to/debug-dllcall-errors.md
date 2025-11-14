# How-To: Debug DllCall Errors

## Problem

Your DllCall is crashing, returning wrong results, or failing silently. You need a systematic process to identify and fix the issue.

## Quick Context

**Common failure modes:**
- Crashes with access violations (bad pointer, wrong type)
- Functions return 0/NULL indicating failure
- ErrorLevel shows -4 (function not found), -3 (DLL problem), -2 (invalid type)
- Results are corrupted or wrong
- Silent failures (function runs but doesn't do what you expect)

**Why DllCall is fragile:** You're bypassing AHK's safety checks and directly interacting with compiled C code. Wrong types, wrong sizes, or invalid pointers cause immediate crashes. Windows APIs assume you know what you're doing.

**The debugging mindset:** Work systematically from simple to complex. Verify each component separately: function exists, types match, parameters correct, buffers allocated, handles valid.

## Key Concepts

### ErrorLevel Values

After DllCall, ErrorLevel indicates what went wrong:

- **Blank/Empty:** Call succeeded or exception occurred
- **-4:** Function not found in DLL (wrong name or DLL)
- **-3:** DLL not accessible (wrong path, bitness, or permissions)
- **-2:** Invalid type name (typo or unquoted in v2)
- **"A-n":** Wrong number of arguments (n bytes missing)

### A_LastError

Contains the Windows error code from GetLastError() called immediately after the function returns:

- **0:** Success (ERROR_SUCCESS)
- **5:** Access denied (ERROR_ACCESS_DENIED)
- **6:** Invalid handle (ERROR_INVALID_HANDLE)
- **87:** Invalid parameter (ERROR_INVALID_PARAMETER)
- **122:** Insufficient buffer (ERROR_INSUFFICIENT_BUFFER)
- **127:** Procedure not found (ERROR_PROC_NOT_FOUND)
- **998:** Invalid access to memory (ERROR_NOACCESS)

### Return Value Checking

Functions signal failure differently:
- BOOL functions: 0 = failure, non-zero = success
- Handle functions: NULL (0) or -1 = failure
- Integer functions: Check MSDN for special error values
- Always check BOTH return value AND A_LastError

## Solution: 10-Step Systematic Debugging Process

### Step 1: Check ErrorLevel and A_LastError

Start with diagnostic output for every DllCall during development:

```ahk
#Requires AutoHotkey v2.0

; Diagnostic wrapper for testing
TestDllCall() {
    result := DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
                     "Int", 0, "Int", 0, "Int", 0, "Int", 0, "UInt", 0x0047)

    MsgBox Format("Result: {1}`nErrorLevel: {2}`nA_LastError: {3}",
                 result, ErrorLevel, A_LastError)
}

; Test it
hwnd := WinExist("A")
TestDllCall()
```

**Interpreting results:**
- ErrorLevel = -4 → Function name is wrong (case matters!)
- ErrorLevel = -3 → DLL path wrong or bitness mismatch
- ErrorLevel = -2 → Type name typo (did you forget quotes in v2?)
- ErrorLevel = "A-n" → Parameter count wrong
- ErrorLevel = blank, result = 0, A_LastError = 0 → Check return type
- ErrorLevel = blank, result = 0, A_LastError != 0 → Real API failure

### Step 2: Verify Function Exists

Use Dependency Walker or DLL Export Viewer to see exact exported names:

```ahk
; Test with known-good function first
MessageBoxWorks := DllCall("MessageBox", "Ptr", 0, "Str", "Test", "Str", "Title", "UInt", 0)
if (ErrorLevel = -4)
    MsgBox "MessageBox not found - something is very wrong"
else
    MsgBox "MessageBox works, problem is with your specific function"

; Verify case sensitivity
result1 := DllCall("User32\SetWindowPos", ...)  ; Wrong case in DLL
result2 := DllCall("user32\SetWindowPos", ...)  ; Wrong case in function
result3 := DllCall("SetWindowPos", ...)         ; Correct
```

**Common mistakes:**
- `setwindowpos` instead of `SetWindowPos` (case matters!)
- `User32\SetWindowPos` instead of `SetWindowPos` (DLL name case sometimes matters)
- Calling ANSI version on Unicode system (`CreateFileA` vs `CreateFileW`)
- Function doesn't exist in older Windows versions

### Step 3: Validate Every Parameter Type

Go through MSDN signature line by line and verify type translation:

```ahk
; MSDN shows:
; BOOL SetWindowPos(
;   HWND hWnd,           → "Ptr"
;   HWND hWndInsertAfter → "Ptr"
;   int X,               → "Int"
;   int Y,               → "Int"
;   int cx,              → "Int"
;   int cy,              → "Int"
;   UINT uFlags          → "UInt"
; );

; Type validation helper
ValidateTypes() {
    hwnd := WinExist("A")

    ; Check each parameter
    MsgBox "hwnd type: " Type(hwnd) " (should be Integer)"
    MsgBox "hwnd value: " hwnd " (should be positive)"

    ; Verify all types are quoted in v2
    try {
        result := DllCall("SetWindowPos", Ptr, hwnd, ...)  ; WRONG - unquoted
    } catch as err {
        MsgBox "Unquoted type error: " err.Message
    }

    result := DllCall("SetWindowPos", "Ptr", hwnd, ...)  ; CORRECT - quoted
}
```

**Type checklist:**
- [ ] All handles/pointers use "Ptr"
- [ ] All types are quoted strings in v2
- [ ] Integer sizes match (Int/UInt = 4 bytes, Int64/UInt64 = 8 bytes)
- [ ] Strings use "Str"/"WStr"/"AStr" for input, "Ptr" for output
- [ ] Output parameters use asterisk: "Type*"
- [ ] Return type specified for pointer-returning functions

### Step 4: Count Parameters

Missing or extra parameters cause "A-n" ErrorLevel:

```ahk
; SetWindowPos needs exactly 7 parameters
CountParameters() {
    hwnd := WinExist("A")

    ; WRONG - only 6 parameters (missing flags)
    result := DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
                     "Int", 0, "Int", 0, "Int", 0, "Int", 0)
    MsgBox "6 params - ErrorLevel: " ErrorLevel  ; Shows "A-4" (missing 4 bytes)

    ; CORRECT - all 7 parameters
    result := DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
                     "Int", 0, "Int", 0, "Int", 0, "Int", 0, "UInt", 0x0047)
    MsgBox "7 params - ErrorLevel: " ErrorLevel  ; Should be blank
}
```

**Parameter counting tips:**
- Count type strings: each "Type" is one parameter
- Check MSDN for optional parameters (can pass 0/NULL)
- Return type doesn't count as a parameter
- Some functions have variable parameters (rare)

### Step 5: Test Minimal Example

Isolate the problem by testing the simplest possible case:

```ahk
; Simplest DllCall that should always work
TestMinimal() {
    ; MessageBox has no complex parameters
    result := DllCall("MessageBox", "Ptr", 0, "Str", "Test",
                     "Str", "Title", "UInt", 0)
    MsgBox "MessageBox result: " result " (should be non-zero)"
}

; Simple version of your function
TestSimpleSetWindowPos() {
    hwnd := WinExist("A")

    ; Simplest call - just show window, change nothing else
    result := DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
                     "Int", 0, "Int", 0, "Int", 0, "Int", 0,
                     "UInt", 0x0047)  ; SWP_NOMOVE|NOSIZE|NOZORDER|SHOWWINDOW

    MsgBox Format("Result: {1}`nError: {2}", result, A_LastError)
}

TestMinimal()
TestSimpleSetWindowPos()
```

**Complexity ladder:**
1. Test MessageBox (always works)
2. Test your function with hardcoded safe values
3. Add one complex parameter at a time
4. Add dynamic values last

### Step 6: Verify Buffer Allocation

Buffer issues are the #1 cause of crashes:

```ahk
; Buffer validation
TestBufferAllocation() {
    ; WRONG - unallocated buffer
    try {
        DllCall("GetWindowText", "Ptr", hwnd, "Ptr", &buffer, "Int", 260)
    } catch as err {
        MsgBox "Crash with unallocated buffer: " err.Message
    }

    ; WRONG - buffer too small
    buffer := Buffer(10, 0)  ; Only 10 bytes
    chars := DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 260, "Int")
    MsgBox "Truncated: " StrGet(buffer)  ; Will be cut off

    ; CORRECT - properly sized buffer
    buffer := Buffer(260 * 2, 0)  ; 260 chars × 2 bytes for UTF-16
    chars := DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 260, "Int")
    MsgBox "Full title: " StrGet(buffer, "UTF-16")
}

hwnd := WinExist("A")
TestBufferAllocation()
```

**Buffer checklist:**
- [ ] Buffer allocated before call: `buffer := Buffer(size, 0)`
- [ ] Size is large enough for output
- [ ] String buffers account for encoding (UTF-16 = chars × 2)
- [ ] Struct sizes account for padding/alignment
- [ ] Buffer.Size checked after allocation
- [ ] Zero-filled with second parameter (0)

### Step 7: Check Handle Validity

Invalid handles cause failures or crashes:

```ahk
; Handle validation
ValidateHandle(hwnd) {
    ; Check if handle exists
    if (!hwnd) {
        MsgBox "Handle is NULL/0"
        return false
    }

    ; Check if handle is -1 (INVALID_HANDLE_VALUE)
    if (hwnd = -1 || hwnd = 0xFFFFFFFF) {
        MsgBox "Handle is INVALID_HANDLE_VALUE"
        return false
    }

    ; Check if window still exists
    if (!DllCall("IsWindow", "Ptr", hwnd)) {
        MsgBox "Window no longer exists"
        return false
    }

    MsgBox "Handle is valid"
    return true
}

; Test it
hwnd := WinExist("A")
ValidateHandle(hwnd)

; Test with invalid handle
ValidateHandle(0)
ValidateHandle(-1)
ValidateHandle(0xDEADBEEF)  ; Random invalid value
```

**Handle validation pattern:**
```ahk
GetWindowHandle(title) {
    hwnd := WinExist(title)

    if (!hwnd) {
        MsgBox "Window not found: " title
        return 0
    }

    if (!DllCall("IsWindow", "Ptr", hwnd)) {
        MsgBox "Window exists in AHK but not in Windows"
        return 0
    }

    return hwnd
}
```

### Step 8: Verify Platform Bitness

32-bit vs 64-bit issues cause crashes:

```ahk
; Platform verification
VerifyPlatform() {
    MsgBox "AHK bitness: " (A_PtrSize = 8 ? "64-bit" : "32-bit")
    MsgBox "Pointer size: " A_PtrSize " bytes"

    ; Test struct size calculation
    ; FLASHWINFO size differs between 32-bit and 64-bit
    structSize := 20 + (A_PtrSize - 4)
    MsgBox "FLASHWINFO size: " structSize " bytes`n(20 on 32-bit, 24 on 64-bit)"
}

VerifyPlatform()

; Platform-safe struct allocation
AllocateFlashWInfo(hwnd) {
    structSize := 20 + (A_PtrSize - 4)
    fwi := Buffer(structSize, 0)

    offset := 0
    NumPut("UInt", structSize, fwi, offset)
    offset += 4
    NumPut("Ptr", hwnd, fwi, offset)      ; Uses A_PtrSize automatically
    offset += A_PtrSize                    ; Skip pointer-sized field
    NumPut("UInt", 0x0F, fwi, offset)

    return fwi
}
```

**Platform checklist:**
- [ ] Use "Ptr" for all handles (not Int/UInt)
- [ ] Struct sizes use A_PtrSize for pointer fields
- [ ] Tested on both 32-bit and 64-bit if distributing
- [ ] DLL bitness matches AHK bitness
- [ ] Offsets account for pointer size differences

### Step 9: Use Try-Catch for Testing

Catch exceptions during development:

```ahk
; Exception-safe testing
SafeDllCall(funcName, params*) {
    try {
        result := DllCall(funcName, params*)
        MsgBox Format("{1} succeeded`nResult: {2}`nErrorLevel: {3}`nA_LastError: {4}",
                     funcName, result, ErrorLevel, A_LastError)
        return result
    } catch as err {
        MsgBox Format("{1} threw exception`nMessage: {2}`nWhat: {3}`nLine: {4}",
                     funcName, err.Message, err.What, err.Line)
        return false
    }
}

; Test it
hwnd := WinExist("A")

; Test with correct call
SafeDllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
           "Int", 0, "Int", 0, "Int", 0, "Int", 0, "UInt", 0x0047)

; Test with wrong type (will throw)
SafeDllCall("SetWindowPos", "Str", hwnd, "Ptr", 0,  ; WRONG - Str instead of Ptr
           "Int", 0, "Int", 0, "Int", 0, "Int", 0, "UInt", 0x0047)
```

**Production error handling:**
```ahk
RobustDllCall(funcName, params*) {
    try {
        result := DllCall(funcName, params*)

        ; Check ErrorLevel
        if (ErrorLevel = -4) {
            throw Error("Function not found: " funcName)
        } else if (ErrorLevel = -3) {
            throw Error("DLL not accessible")
        } else if (ErrorLevel = -2) {
            throw Error("Invalid type in parameters")
        } else if (InStr(ErrorLevel, "A-")) {
            throw Error("Wrong parameter count: " ErrorLevel)
        }

        ; Check return value (for BOOL functions)
        if (!result && A_LastError != 0) {
            throw Error(Format("{1} failed with error {2}", funcName, A_LastError))
        }

        return result

    } catch as err {
        MsgBox "DllCall error: " err.Message
        return false
    }
}
```

### Step 10: Search for Working Examples

Learn from others' implementations:

```ahk
; Search strategy:
; 1. Google: "SetWindowPos autohotkey v2"
; 2. Check AutoHotkey forums
; 3. Look at GitHub repositories
; 4. Check pinvoke.net for C# equivalents
; 5. Read MSDN examples in C/C++

; Compare your implementation
YourImplementation() {
    hwnd := WinExist("A")
    result := DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
                     "Int", 0, "Int", 0, "Int", 0, "Int", 0, "UInt", 0x0047)
    return result
}

; Working example from forums/documentation
WorkingImplementation() {
    hwnd := WinExist("A")
    result := DllCall("User32.dll\SetWindowPos", "Ptr", hwnd, "Ptr", 0,
                     "Int", 0, "Int", 0, "Int", 0, "Int", 0, "UInt", 0x0047)
    return result
}

; Compare side-by-side
MsgBox "Your result: " YourImplementation() " vs Working result: " WorkingImplementation()
```

## Complete Debugging Example

Here's a full example showing systematic debugging of a failing DllCall:

```ahk
#Requires AutoHotkey v2.0

; Broken version - find the bugs!
BrokenGetWindowTitle(hwnd) {
    buffer := Buffer(260, 0)
    length := DllCall("GetWindowText", "Ptr", hwnd,
                     "Ptr", buffer, "Int", 260)
    return StrGet(buffer)
}

; Debugging process
DebugGetWindowTitle() {
    hwnd := WinExist("A")

    ; Step 1: Check ErrorLevel and A_LastError
    MsgBox "=== Step 1: Basic Diagnostics ==="
    buffer := Buffer(260, 0)
    length := DllCall("GetWindowText", "Ptr", hwnd,
                     "Ptr", buffer, "Int", 260, "Int")
    MsgBox Format("Length: {1}`nErrorLevel: {2}`nA_LastError: {3}",
                 length, ErrorLevel, A_LastError)

    ; Step 2: Verify function exists
    MsgBox "=== Step 2: Function Exists ==="
    try {
        result := DllCall("GetWindowText", "Ptr", 0, "Ptr", 0, "Int", 0, "Int")
        MsgBox "Function exists (ErrorLevel: " ErrorLevel ")"
    } catch as err {
        MsgBox "Function problem: " err.Message
    }

    ; Step 3: Validate types
    MsgBox "=== Step 3: Type Validation ==="
    MsgBox "Param 1: Ptr " hwnd " (" Type(hwnd) ")"
    MsgBox "Param 2: Ptr to Buffer"
    MsgBox "Param 3: Int 260"
    MsgBox "Return: Int"

    ; Step 4: Count parameters (3 + return = correct)
    MsgBox "=== Step 4: Parameter Count ==="
    MsgBox "Parameters: 3 (hwnd, buffer, length) + return type"

    ; Step 5: Test minimal
    MsgBox "=== Step 5: Minimal Test ==="
    result := DllCall("MessageBox", "Ptr", 0, "Str", "Works?", "Str", "Test", "UInt", 0)

    ; Step 6: Verify buffer
    MsgBox "=== Step 6: Buffer Verification ==="
    testBuffer := Buffer(260 * 2, 0)
    MsgBox "Buffer allocated: " testBuffer.Size " bytes"

    ; Step 7: Check handle
    MsgBox "=== Step 7: Handle Validation ==="
    if (!hwnd)
        MsgBox "ERROR: Invalid handle"
    else if (!DllCall("IsWindow", "Ptr", hwnd))
        MsgBox "ERROR: Window doesn't exist"
    else
        MsgBox "Handle is valid"

    ; Step 8: Platform check
    MsgBox "=== Step 8: Platform ==="
    MsgBox "Bitness: " (A_PtrSize = 8 ? "64-bit" : "32-bit")

    ; Step 9: Try-catch
    MsgBox "=== Step 9: Exception Test ==="
    try {
        buffer := Buffer(260 * 2, 0)
        length := DllCall("GetWindowText", "Ptr", hwnd,
                         "Ptr", buffer, "Int", 260, "Int")
        title := StrGet(buffer, "UTF-16")
        MsgBox "Success! Title: " title
    } catch as err {
        MsgBox "Exception: " err.Message
    }
}

; Run the debugging process
DebugGetWindowTitle()

; Fixed version
FixedGetWindowTitle(hwnd) {
    ; Get required length
    length := DllCall("GetWindowTextLength", "Ptr", hwnd, "Int") + 1

    if (length <= 1)
        return ""

    ; Allocate proper buffer size (characters × 2 for UTF-16)
    buffer := Buffer(length * 2, 0)

    ; Get title with return type
    chars := DllCall("GetWindowText", "Ptr", hwnd,
                    "Ptr", buffer, "Int", length, "Int")

    if (!chars && A_LastError != 0) {
        MsgBox "GetWindowText failed: " A_LastError
        return ""
    }

    ; Extract with correct encoding
    return StrGet(buffer, "UTF-16")
}

; Test fixed version
hwnd := WinExist("A")
title := FixedGetWindowTitle(hwnd)
MsgBox "Window title: " title
```

## Common Gotchas Reference

Quick reference for the most common mistakes:

```ahk
; GOTCHA 1: Unquoted types in v2
DllCall("Func", UInt, value)           ; WRONG - v2 error
DllCall("Func", "UInt", value)         ; CORRECT - quoted

; GOTCHA 2: Wrong case function name
DllCall("setwindowpos", ...)           ; WRONG - returns ErrorLevel -4
DllCall("SetWindowPos", ...)           ; CORRECT - exact case

; GOTCHA 3: Int instead of Ptr for handles
DllCall("Func", "Int", hwnd, ...)      ; WRONG - breaks on 64-bit
DllCall("Func", "Ptr", hwnd, ...)      ; CORRECT - platform-independent

; GOTCHA 4: Missing return type for pointers
handle := DllCall("LoadLibrary", "Str", "user32.dll")           ; WRONG - truncated
handle := DllCall("LoadLibrary", "Str", "user32.dll", "Ptr")    ; CORRECT

; GOTCHA 5: No buffer allocation
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", &buffer, "Int", 260)  ; WRONG - crash
buffer := Buffer(520, 0)                                           ; CORRECT
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 260)

; GOTCHA 6: Forgot VarRef & in v2
DllCall("GetWindowThreadProcessId", "Ptr", hwnd, "UInt*", processId)    ; WRONG
DllCall("GetWindowThreadProcessId", "Ptr", hwnd, "UInt*", &processId)   ; CORRECT

; GOTCHA 7: Wrong buffer size for strings
buffer := Buffer(260, 0)                    ; WRONG - bytes not chars
buffer := Buffer(260 * 2, 0)                ; CORRECT - chars × 2 for UTF-16

; GOTCHA 8: Didn't check return value
DllCall("SomeFunction", ...)
result := StrGet(buffer)                    ; WRONG - use even if call failed
if (DllCall("SomeFunction", ...))           ; CORRECT - check first
    result := StrGet(buffer)

; GOTCHA 9: Struct size not set
info := Buffer(40, 0)                       ; WRONG - cbSize field is 0
DllCall("GetMonitorInfo", "Ptr", hMon, "Ptr", info)

info := Buffer(40, 0)                       ; CORRECT - set cbSize
NumPut("UInt", 40, info, 0)
DllCall("GetMonitorInfo", "Ptr", hMon, "Ptr", info)

; GOTCHA 10: Forgot StrGet after buffer fill
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 260)
MsgBox buffer                               ; WRONG - shows buffer object
MsgBox StrGet(buffer, "UTF-16")             ; CORRECT - extract string
```

## Cross-References

For complete type system reference: docs/manuals/ahk2/reference/dllcall-types.md
For working with structs: docs/manuals/ahk2/how-to/work-with-structs.md
For output parameters and buffers: docs/manuals/ahk2/how-to/dllcall-output-parameters.md
For understanding error codes: docs/manuals/ahk2/reference/windows-error-codes.md
For MSDN documentation translation: docs/manuals/ahk2/explanation/windows-api-translation.md
