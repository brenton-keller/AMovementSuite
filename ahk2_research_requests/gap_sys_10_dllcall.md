# AHK2 Research Request: DllCall Deep Understanding

## What I Need to Learn

My codebase has ~78 DllCall instances but I'm just pattern-copying without understanding. This mysterious call appears everywhere:
```ahk
DllCall("SetWindowPos", "Ptr", mwin, "Ptr", 0, "Int", 0, "Int", 0,
        "Int", 0, "Int", 0, "UInt", 0x0001|0x0002|0x0004|0x0400)
```

**What do these flags mean? Why this combination? When would I use different flags?**

I need to understand DllCall deeply enough to:
- Read Windows API documentation and translate to AHK2
- Debug when DllCalls fail silently
- Know when to use DllCall vs built-in functions

## My Current Understanding (Challenge This!)

I believe:
- DllCall is for calling C functions in DLLs
- "Ptr" means pointer, "Int" means integer
- Return value is the function result
- DllCall throws error on failure

**What am I missing or wrong about?**

## The Mystery I Need Solved

```ahk
DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0,
        "Int", 0, "Int", 0, "Int", 0, "Int", 0,
        "UInt", 0x0001|0x0002|0x0004|0x0400)
```

Explain EVERY part:
- Why "Ptr" for hwnd but "Int" for coordinates?
- What is the second "Ptr", 0 parameter?
- What do these flags mean: 0x0001, 0x0002, 0x0004, 0x0400?
- Why OR them together?
- When would I use different flags?
- Why is return value not checked?

## Specific Questions

1. **Type Marshaling**: How does AHK2 convert types to C types? What's "Ptr" vs "Int" vs "UInt" vs "Str"?

2. **Return Values**: How do I read return values? What if function returns struct?

3. **Output Parameters**: How do I pass buffers to fill? What's the pattern for output strings?

4. **Error Handling**: When does DllCall throw vs return error code vs fail silently?

5. **Common Windows APIs**: Which DLLs/functions are most useful? When to use vs AHK2 built-ins?

## How to Write This Guide

### 1. Start with "The Stack"

Show what happens at memory level:
```
AHK2 code:
  DllCall("SetWindowPos", "Ptr", hwnd, "Int", x)

Stack before call:
┌────────────┐
│   hwnd     │ ← Pointer (8 bytes on 64-bit)
│  (ptr)     │
├────────────┤
│     x      │ ← Integer (4 bytes)
│   (int)    │
└────────────┘

C function signature:
  BOOL SetWindowPos(HWND hwnd, HWND insertAfter, int X, int Y, ...)

Stack marshaling:
  AHK2 "Ptr" → C HWND (both 8-byte pointers)
  AHK2 "Int" → C int (both 4-byte integers)
```

### 2. Type Conversion Table

Complete reference:

| AHK2 Type | C Type | Size | Use For |
|-----------|--------|------|---------|
| Int | int | 4 bytes | Signed integers |
| UInt | unsigned int | 4 bytes | Unsigned integers, flags |
| Int64 | __int64 | 8 bytes | Large integers |
| Ptr | void* | 4/8 bytes | Pointers, handles |
| Str | wchar_t* | Variable | Strings (UTF-16) |
| AStr | char* | Variable | ANSI strings |
| Float | float | 4 bytes | Single precision |
| Double | double | 8 bytes | Double precision |

Include common mistakes for each type.

### 3. Decode SetWindowPos Completely

Take the mystery call from my code and dissect it:

```ahk
DllCall("SetWindowPos",
    "Ptr", hwnd,        // HWND hWnd - window handle
    "Ptr", 0,           // HWND hWndInsertAfter - 0 means keep current Z-order
    "Int", 0,           // int X - new X position (ignored with SWP_NOMOVE)
    "Int", 0,           // int Y - new Y position (ignored with SWP_NOMOVE)
    "Int", 0,           // int cx - new width (ignored with SWP_NOSIZE)
    "Int", 0,           // int cy - new height (ignored with SWP_NOSIZE)
    "UInt", 0x0001|0x0002|0x0004|0x0400)
    //      ^^^^^ Flags - let's decode:
```

Decode the flags:
```
0x0001 = SWP_NOSIZE      → Don't change size
0x0002 = SWP_NOMOVE      → Don't change position
0x0004 = SWP_NOZORDER    → Don't change Z-order
0x0400 = SWP_NOSENDCHANGING → Don't send WM_WINDOWPOSCHANGING

Combined: 0x0407 = Update window without changing size/pos/Z, no message

PURPOSE: Forces window redraw/update without actually moving it
```

Then explain WHEN and WHY you'd use this combination.

### 4. Common DllCall Patterns

**Pattern 1: Getting window info**
```ahk
; GetWindowRect - fill a RECT struct
VarSetStrCapacity(&RECT, 16)  ; 4 ints = 16 bytes
DllCall("GetWindowRect", "Ptr", hwnd, "Ptr", &RECT)

; Now decode RECT:
left   := NumGet(&RECT, 0, "Int")
top    := NumGet(&RECT, 4, "Int")
right  := NumGet(&RECT, 8, "Int")
bottom := NumGet(&RECT, 12, "Int")

// Explain EVERY detail of this pattern
```

**Pattern 2: Output strings**
```ahk
; GetWindowText - fill buffer with window title
bufSize := 256
VarSetStrCapacity(&buf, bufSize)
length := DllCall("GetWindowText", "Ptr", hwnd, "Str", &buf, "Int", bufSize)
windowTitle := StrGet(&buf)

// Why StrGet? When is it needed?
```

**Pattern 3: Checking return values**
```ahk
; Function that returns success/failure
result := DllCall("ShowWindow", "Ptr", hwnd, "Int", SW_SHOW)
if (!result)
    MsgBox "ShowWindow failed"

// When to check return value vs when to ignore?
```

### 5. When to Use DllCall vs Built-ins

Create decision table:

| Task | AHK2 Built-in | DllCall Alternative | When to Use DllCall |
|------|---------------|---------------------|---------------------|
| Move window | WinMove | SetWindowPos | Need fine control over flags |
| Get window pos | WinGetPos | GetWindowRect | Never (built-in better) |
| Set topmost | WinSetAlwaysOnTop | SetWindowPos | Never (built-in better) |
| Flash window | N/A | FlashWindowEx | No built-in exists |

### 6. Debugging Failed DllCalls

Step-by-step debugging guide:
```ahk
; DllCall failed - how to debug:

Step 1: Check return value
result := DllCall("FunctionName", ...)
if (result = 0 || result = -1)
    MsgBox "Failed"

Step 2: Get last error
errorCode := A_LastError
MsgBox "Error code: " errorCode

Step 3: Look up error code
// Where to find error code meanings?

Step 4: Verify parameter types
// How to verify types match C signature?

Step 5: Check DLL is loaded
// Is user32.dll available? Is function name correct?
```

### 7. Common API Functions Reference

For each, show:
- C signature
- AHK2 DllCall translation
- When to use
- Example

**user32.dll functions:**
- SetWindowPos
- GetWindowRect
- ShowWindow
- SetForegroundWindow
- FlashWindowEx

**kernel32.dll functions:**
- GetLastError
- Sleep (vs AHK2 Sleep)
- CreateFile
- ReadFile/WriteFile

### 8. The Gotchas (From Pain)

**"Why doesn't my DllCall work?"**
- Function name is case-sensitive in some DLLs
- 32-bit vs 64-bit calling conventions differ
- Some functions have A/W suffix (ANSI/Wide)
- Null vs 0 vs "" are different for pointers

**"Why does it work sometimes but not always?"**
- Buffer size too small (string truncation)
- Struct alignment issues
- Thread safety (calling from different threads)

**"Why does my script crash?"**
- Bad pointer (accessing invalid memory)
- Buffer overflow
- Calling convention mismatch (cdecl vs stdcall)

### 9. Struct Handling Master Class

Show complete pattern:
```ahk
; Define struct
RECT := Buffer(16)  ; 4 ints * 4 bytes each

; Fill struct
NumPut("Int", 100, RECT, 0)   ; left
NumPut("Int", 100, RECT, 4)   ; top
NumPut("Int", 200, RECT, 8)   ; right
NumPut("Int", 200, RECT, 12)  ; bottom

; Pass to function
DllCall("SomeFunction", "Ptr", RECT)

; Read back
left := NumGet(RECT, 0, "Int")
```

Explain Buffer vs VarSetCapacity vs VarSetStrCapacity.

## Success Criteria

After this guide:
1. ✓ Can read MSDN documentation and translate to DllCall
2. ✓ Understand what the mystery SetWindowPos call does and why
3. ✓ Debug failed DllCall systematically
4. ✓ Handle structs (pass in, read out)
5. ✓ Know when to use DllCall vs built-in functions
6. ✓ Never crash from bad DllCall parameters

## Most Important

**Teach me to read Windows API documentation** and confidently translate C signatures to AHK2 DllCall syntax.

I want to go from "copying DllCall examples" to "reading MSDN and implementing any Windows API myself."
