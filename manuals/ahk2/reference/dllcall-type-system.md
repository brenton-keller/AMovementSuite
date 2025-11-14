# DllCall Type System Reference

Comprehensive lookup tables for AutoHotkey v2 DllCall type specifications, Windows API translations, and common patterns.

## Complete Type Reference Table

| AHK Type | C Equivalent | Size (bytes) | Range/Usage | Common Mistakes |
|----------|--------------|--------------|-------------|-----------------|
| **Char** | char | 1 | -128 to 127 | Using for byte values (use UChar) |
| **UChar** | unsigned char | 1 | 0 to 255 | Forgetting it's unsigned |
| **Short** | short | 2 | -32,768 to 32,767 | Confusing with UShort |
| **UShort** | unsigned short, WORD | 2 | 0 to 65,535 | Using for WORD in structs |
| **Int** | int, LONG, BOOL | 4 | -2.1B to 2.1B | Using for handles on 64-bit |
| **UInt** | unsigned int, DWORD | 4 | 0 to 4.3B | Using for pointers |
| **Int64** | __int64, LONGLONG | 8 | ±9.2×10^18 | Forgetting to specify as return type |
| **UInt64** | unsigned __int64 | 8 | 0 to 1.8×10^19 | Using when Ptr is appropriate |
| **Ptr** | void*, HANDLE, HWND | 4 or 8 | Platform-dependent | **CRITICAL: Use for ALL handles/pointers** |
| **UPtr** | size_t, ULONG_PTR | 4 or 8 | Unsigned pointer-sized | Confusing with Ptr |
| **Float** | float | 4 | ~7 digits precision | Using for doubles |
| **Double** | double | 8 | ~15 digits precision | Default for AHK floats |
| **Str** | LPTSTR (UTF-16 in v2) | N/A | Native strings | Using for output buffers |
| **WStr** | LPWSTR, wchar_t* | N/A | Unicode strings | Using for output buffers |
| **AStr** | LPSTR, char* | N/A | ANSI strings | Creates temp conversion in v2 |

**Critical Rules:**
- Use `Ptr` for ALL handles (HWND, HANDLE, HDC) and pointers
- Never use Int/UInt for handles on 64-bit systems
- Integer types: signed = Int variants, unsigned = UInt variants
- String types for INPUT only; use Ptr + Buffer for OUTPUT

## Windows API Type Translation

Complete mapping of Windows types to AHK DllCall types.

| Windows Type | What It Is | AHK Type | Notes |
|--------------|------------|----------|-------|
| **BOOL, BOOLEAN** | int (0 or 1) | Int | Check for nonzero, not just 1 |
| **BYTE** | unsigned char | UChar | 0-255 |
| **WORD** | unsigned short | UShort | 16-bit unsigned |
| **DWORD, ULONG** | unsigned long | UInt | 32-bit unsigned |
| **LONG** | long | Int | 32-bit signed |
| **LONGLONG** | __int64 | Int64 | 64-bit signed |
| **UINT_PTR, SIZE_T** | pointer-sized unsigned | UPtr | Use for sizes |
| **INT_PTR, SSIZE_T** | pointer-sized signed | Ptr | Rare, usually pointers |
| **HANDLE** | void* (generic handle) | Ptr | All handle types |
| **HWND** | window handle | Ptr | Window handles |
| **HDC** | device context handle | Ptr | Graphics contexts |
| **HBITMAP** | bitmap handle | Ptr | Bitmap resources |
| **HINSTANCE, HMODULE** | module handle | Ptr | DLL/EXE handles |
| **LPVOID, PVOID** | void* | Ptr | Generic pointers |
| **LPSTR, LPCSTR** | char* | AStr or Ptr | ANSI strings |
| **LPWSTR, LPCWSTR** | wchar_t* | WStr or Ptr | Unicode strings |
| **LPTSTR, LPCTSTR** | auto char/wchar_t* | Str or Ptr | Auto-select encoding |

**Translation Pattern:**
1. Any handle (H prefix) → `Ptr`
2. Any pointer (LP/P prefix) → `Ptr`
3. 32-bit integers → `Int` (signed) or `UInt` (unsigned)
4. 64-bit integers → `Int64` or `UInt64`
5. Pointer-sized values → `Ptr` or `UPtr`

## Type Prefix Quick Reference

Understanding Windows type naming conventions.

| Prefix/Suffix | Meaning | Example | AHK Type |
|---------------|---------|---------|----------|
| **H** | Handle | HWND, HDC, HBITMAP | Ptr |
| **LP** | Long Pointer (far pointer) | LPVOID, LPSTR | Ptr |
| **P** | Pointer | PVOID, PCHAR | Ptr |
| **C** | Const (read-only) | LPCSTR, LPCWSTR | Ptr (still pointer) |
| **W** | Wide (Unicode) | LPWSTR, CreateFileW | WStr or Ptr |
| **A** | ANSI (single-byte) | LPSTR, CreateFileA | AStr or Ptr |
| **DW** | Double Word (32-bit) | DWORD | UInt |
| **Q** | Quad Word (64-bit) | QWORD | UInt64 |

**Common Combinations:**
- `LPCSTR` = Long Pointer to Const String (ANSI) → `"AStr"` or `"Ptr"`
- `LPCWSTR` = Long Pointer to Const Wide String (Unicode) → `"WStr"` or `"Ptr"`
- `LPCTSTR` = Long Pointer to Const T String (auto) → `"Str"` or `"Ptr"`

## String Type Comparison

| Type | Encoding | Direction | Use Case | Buffer Handling |
|------|----------|-----------|----------|-----------------|
| **Str** | UTF-16 (native) | Input | Default string type | Passes string address directly |
| **WStr** | UTF-16 (explicit) | Input | Force Unicode API | Passes wide string address |
| **AStr** | ANSI (CP0) | Input | Legacy ANSI APIs | Creates temp conversion |
| **Ptr + Buffer** | Any | Output | Function fills buffer | Pre-allocate with Buffer() |

**Input String Example:**
```ahk
; All valid for input strings
DllCall("MessageBox", "Ptr", 0, "Str", "Message", "Str", "Title", "UInt", 0)
DllCall("MessageBoxW", "Ptr", 0, "WStr", "Message", "WStr", "Title", "UInt", 0)
DllCall("MessageBoxA", "Ptr", 0, "AStr", "Message", "AStr", "Title", "UInt", 0)
```

**Output String Example:**
```ahk
; WRONG - crashes
DllCall("GetWindowText", "Ptr", hwnd, "Str", buffer, "Int", 260)

; CORRECT - pre-allocate buffer
buffer := Buffer(260 * 2, 0)  ; 260 chars × 2 bytes for UTF-16
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 260)
title := StrGet(buffer, "UTF-16")
```

**String Extraction Functions:**
- `StrGet(buffer, encoding)` - Extract string from buffer
- `StrGet(buffer, length, encoding)` - Extract limited length
- `StrPut(string, buffer, encoding)` - Write string to buffer

**Common Encodings:**
- `"UTF-16"` or `"UTF-16 LE"` - Windows Unicode (most common)
- `"UTF-8"` - UTF-8 encoded strings
- `"CP0"` - ANSI (current code page)

## Output Parameter Notation

Passing variable addresses for functions to write results.

| Notation | Meaning | v2 Syntax | Example |
|----------|---------|-----------|---------|
| **Type\*** | Pointer to type | `"Type*", &var` | `"Int*", &result` |
| **TypeP** | Pointer to type (alternate) | `"TypeP", &var` | `"IntP", &result` |
| **Ptr** | Generic pointer | `"Ptr", buffer` | `"Ptr", Buffer(16)` |

**Output Parameter Examples:**

```ahk
; Single output parameter
processId := 0
threadId := DllCall("GetWindowThreadProcessId", "Ptr", hwnd, "UInt*", &processId, "UInt")
; threadId = return value, processId filled via pointer

; Multiple output parameters
x := 0, y := 0, w := 0, h := 0
DllCall("GetWindowRect", "Ptr", hwnd, "Int*", &x, "Int*", &y, "Int*", &w, "Int*", &h)
```

**Buffer Pattern (Struct Output):**

```ahk
; Allocate buffer for struct
rect := Buffer(16, 0)  ; 4 Ints × 4 bytes = 16 bytes

; Pass buffer address
DllCall("GetWindowRect", "Ptr", hwnd, "Ptr", rect)

; Extract values with NumGet
left := NumGet(rect, 0, "Int")
top := NumGet(rect, 4, "Int")
right := NumGet(rect, 8, "Int")
bottom := NumGet(rect, 12, "Int")
```

**VarRef Requirement in v2:**
- v1: `DllCall("Function", "Int*", myVar)` ❌ (wrong in v2)
- v2: `DllCall("Function", "Int*", &myVar)` ✅ (correct)

## Return Type Specifications

Specifying what the function returns.

| Return Type | When to Specify | Default Behavior | Example |
|-------------|-----------------|------------------|---------|
| **Int** | Default | Assumes Int return | `DllCall("SetWindowPos", ...)` |
| **Ptr** | Handle/pointer returns | Would truncate on 64-bit | `DllCall("LoadLibrary", "Str", dll, "Ptr")` |
| **Int64** | 64-bit integer returns | Would truncate to 32-bit | `DllCall("GetFileSize", ..., "Int64")` |
| **UInt** | Unsigned 32-bit | Interprets as unsigned | `DllCall("GetTickCount", "UInt")` |
| **Float/Double** | Floating-point returns | Would misinterpret bits | `DllCall("SomeFloatFunc", ..., "Double")` |
| **Str** | Direct string returns | Rare in Windows API | `DllCall("SomeFunc", "Str")` |

**Return Type Syntax:**
```ahk
; Return type as final parameter
result := DllCall("FunctionName", "Type1", param1, "Type2", param2, "ReturnType")
```

**Critical Cases:**

```ahk
; WRONG - truncates 64-bit handle to 32-bit
hModule := DllCall("LoadLibrary", "Str", "user32.dll")

; CORRECT - preserves full 64-bit pointer
hModule := DllCall("LoadLibrary", "Str", "user32.dll", "Ptr")

; WRONG - truncates 64-bit file size
size := DllCall("GetFileSize", "Ptr", hFile, "Ptr", 0)

; CORRECT - gets full 64-bit size
size := DllCall("GetFileSize", "Ptr", hFile, "Ptr", 0, "Int64")
```

**Default Return Type:**
If omitted, DllCall assumes `"Int"` return type. This works for:
- BOOL returns (nonzero = success)
- Integer status codes
- 32-bit values on any platform

## Common Windows API Constants

Frequently used constant values organized by API.

### SetWindowPos Flags

| Flag | Hex Value | Decimal | Effect |
|------|-----------|---------|--------|
| SWP_NOSIZE | 0x0001 | 1 | Retains current size |
| SWP_NOMOVE | 0x0002 | 2 | Retains current position |
| SWP_NOZORDER | 0x0004 | 4 | Retains current Z-order |
| SWP_NOREDRAW | 0x0008 | 8 | Doesn't redraw changes |
| SWP_NOACTIVATE | 0x0010 | 16 | Doesn't activate window |
| SWP_FRAMECHANGED | 0x0020 | 32 | Applies new frame styles |
| SWP_SHOWWINDOW | 0x0040 | 64 | Displays the window |
| SWP_HIDEWINDOW | 0x0080 | 128 | Hides the window |
| SWP_NOCOPYBITS | 0x0100 | 256 | Discards client area |
| SWP_NOOWNERZORDER | 0x0200 | 512 | Doesn't change owner Z-order |
| SWP_NOSENDCHANGING | 0x0400 | 1024 | Prevents WM_WINDOWPOSCHANGING |
| SWP_ASYNCWINDOWPOS | 0x4000 | 16384 | Posts request to window thread |

**Common Combinations:**
```ahk
; Show window without moving/resizing
0x0001|0x0002|0x0040  ; SWP_NOSIZE|NOMOVE|SHOWWINDOW

; Make topmost without moving/resizing
0x0001|0x0002  ; SWP_NOSIZE|NOMOVE (with hWndInsertAfter = -1)

; Update frame after style change
0x0001|0x0002|0x0004|0x0020  ; NOSIZE|NOMOVE|NOZORDER|FRAMECHANGED
```

### SetWindowPos Z-Order Values

| Value | Name | Effect |
|-------|------|--------|
| -1 | HWND_TOPMOST | Always on top |
| -2 | HWND_NOTOPMOST | Remove always-on-top |
| 0 | HWND_TOP | Top of Z-order (not topmost) |
| 1 | HWND_BOTTOM | Bottom of Z-order |

### ShowWindow Commands

| Value | Name | Effect |
|-------|------|--------|
| 0 | SW_HIDE | Hides the window |
| 1 | SW_SHOWNORMAL | Activates and displays normal |
| 2 | SW_SHOWMINIMIZED | Activates and minimizes |
| 3 | SW_SHOWMAXIMIZED | Activates and maximizes |
| 4 | SW_SHOWNOACTIVATE | Displays in recent size/position |
| 5 | SW_SHOW | Activates and displays current |
| 6 | SW_MINIMIZE | Minimizes, activates next window |
| 7 | SW_SHOWMINNOACTIVE | Minimizes, doesn't activate |
| 8 | SW_SHOWNA | Displays current, doesn't activate |
| 9 | SW_RESTORE | Activates and restores |
| 10 | SW_SHOWDEFAULT | Sets show state per STARTUPINFO |

### CreateFile Access Modes

| Flag | Hex Value | Meaning |
|------|-----------|---------|
| GENERIC_READ | 0x80000000 | Read access |
| GENERIC_WRITE | 0x40000000 | Write access |
| GENERIC_EXECUTE | 0x20000000 | Execute access |
| GENERIC_ALL | 0x10000000 | All access |

### CreateFile Share Modes

| Flag | Hex Value | Meaning |
|------|-----------|---------|
| FILE_SHARE_READ | 0x00000001 | Allow others to read |
| FILE_SHARE_WRITE | 0x00000002 | Allow others to write |
| FILE_SHARE_DELETE | 0x00000004 | Allow others to delete |

### CreateFile Creation Disposition

| Value | Name | Behavior |
|-------|------|----------|
| 1 | CREATE_NEW | Fails if exists |
| 2 | CREATE_ALWAYS | Overwrites if exists |
| 3 | OPEN_EXISTING | Fails if doesn't exist |
| 4 | OPEN_ALWAYS | Creates if doesn't exist |
| 5 | TRUNCATE_EXISTING | Opens and clears if exists |

### MonitorFromWindow Flags

| Value | Name | Effect |
|-------|------|--------|
| 0x0 | MONITOR_DEFAULTTONULL | Return NULL |
| 0x1 | MONITOR_DEFAULTTOPRIMARY | Return primary monitor |
| 0x2 | MONITOR_DEFAULTTONEAREST | Return nearest monitor |

## Struct Size Quick Reference

Common Windows structures with field offsets.

### RECT (16 bytes)

```
Offset | Field  | Type | Size
-------|--------|------|-----
0      | left   | Int  | 4
4      | top    | Int  | 4
8      | right  | Int  | 4
12     | bottom | Int  | 4
```

**Usage:**
```ahk
rect := Buffer(16, 0)
DllCall("GetWindowRect", "Ptr", hwnd, "Ptr", rect)
left := NumGet(rect, 0, "Int")
top := NumGet(rect, 4, "Int")
right := NumGet(rect, 8, "Int")
bottom := NumGet(rect, 12, "Int")
```

### POINT (8 bytes)

```
Offset | Field | Type | Size
-------|-------|------|-----
0      | x     | Int  | 4
4      | y     | Int  | 4
```

**Usage:**
```ahk
pt := Buffer(8, 0)
DllCall("GetCursorPos", "Ptr", pt)
x := NumGet(pt, 0, "Int")
y := NumGet(pt, 4, "Int")
```

### SYSTEMTIME (16 bytes)

```
Offset | Field         | Type   | Size
-------|---------------|--------|-----
0      | wYear         | UShort | 2
2      | wMonth        | UShort | 2
4      | wDayOfWeek    | UShort | 2
6      | wDay          | UShort | 2
8      | wHour         | UShort | 2
10     | wMinute       | UShort | 2
12     | wSecond       | UShort | 2
14     | wMilliseconds | UShort | 2
```

**Usage:**
```ahk
st := Buffer(16, 0)
DllCall("GetSystemTime", "Ptr", st)
year := NumGet(st, 0, "UShort")
month := NumGet(st, 2, "UShort")
day := NumGet(st, 6, "UShort")
hour := NumGet(st, 8, "UShort")
minute := NumGet(st, 10, "UShort")
second := NumGet(st, 12, "UShort")
```

### MONITORINFO (40 bytes)

```
Offset | Field        | Type  | Size
-------|--------------|-------|-----
0      | cbSize       | UInt  | 4
4      | rcMonitor    | RECT  | 16
20     | rcWork       | RECT  | 16
36     | dwFlags      | UInt  | 4
```

**Usage (cbSize initialization required):**
```ahk
info := Buffer(40, 0)
NumPut("UInt", 40, info, 0)  ; CRITICAL: Set cbSize
DllCall("GetMonitorInfo", "Ptr", hMonitor, "Ptr", info)

; rcMonitor (offset 4)
monLeft := NumGet(info, 4, "Int")
monTop := NumGet(info, 8, "Int")
monRight := NumGet(info, 12, "Int")
monBottom := NumGet(info, 16, "Int")

; rcWork (offset 20)
workLeft := NumGet(info, 20, "Int")
workTop := NumGet(info, 24, "Int")
workRight := NumGet(info, 28, "Int")
workBottom := NumGet(info, 32, "Int")

; dwFlags (offset 36)
isPrimary := NumGet(info, 36, "UInt") & 1
```

### FLASHWINFO (20 + A_PtrSize bytes)

```
Offset         | Field     | Type  | Size
---------------|-----------|-------|-------------
0              | cbSize    | UInt  | 4
4              | hwnd      | Ptr   | 4 or 8
4 + A_PtrSize  | dwFlags   | UInt  | 4
8 + A_PtrSize  | uCount    | UInt  | 4
12 + A_PtrSize | dwTimeout | UInt  | 4
```

**Usage:**
```ahk
structSize := 20 + A_PtrSize
fwi := Buffer(structSize, 0)

offset := 0
NumPut("UInt", structSize, fwi, offset)
offset += 4
NumPut("Ptr", hwnd, fwi, offset)
offset += A_PtrSize
NumPut("UInt", 0x0F, fwi, offset)  ; FLASHW_ALL|TIMERNOFG
offset += 4
NumPut("UInt", 5, fwi, offset)     ; Flash count
offset += 4
NumPut("UInt", 0, fwi, offset)     ; Timeout

DllCall("FlashWindowEx", "Ptr", fwi)
```

## ErrorLevel Values Reference

DllCall sets ErrorLevel to indicate call status.

| ErrorLevel | Meaning | Cause | Solution |
|------------|---------|-------|----------|
| **-4** | Function not found | Wrong name, wrong DLL | Check spelling, case-sensitivity |
| **-3** | DLL not accessible | DLL missing, wrong bitness | Verify path, match AHK bitness |
| **-2** | Invalid type | Type name typo, unquoted in v2 | Quote type strings in v2 |
| **A-n** | Wrong argument count | Missing/extra parameters | Count against MSDN signature |
| **0** (blank) | Exception occurred | Access violation, bad pointer | Validate parameters, check types |
| **(empty)** | Call succeeded | Normal execution | Check return value and A_LastError |

**Error Check Pattern:**
```ahk
result := DllCall("FunctionName", ...)
if (ErrorLevel != "") {
    MsgBox "DllCall failed with ErrorLevel: " ErrorLevel
    return
}
```

**ErrorLevel "A-n" Interpretation:**
- "A-4" means missing 4 bytes of parameters
- On 64-bit: missing 1 pointer = "A-8"
- On 32-bit: missing 1 int = "A-4"
- Indicates parameter count mismatch

## A_LastError Common Codes

Windows GetLastError() values captured in A_LastError.

| Code | Name | Meaning |
|------|------|---------|
| 0 | ERROR_SUCCESS | Operation succeeded |
| 2 | ERROR_FILE_NOT_FOUND | File not found |
| 3 | ERROR_PATH_NOT_FOUND | Path not found |
| 5 | ERROR_ACCESS_DENIED | Insufficient permissions |
| 6 | ERROR_INVALID_HANDLE | Handle is invalid or closed |
| 8 | ERROR_NOT_ENOUGH_MEMORY | Memory allocation failed |
| 87 | ERROR_INVALID_PARAMETER | Wrong parameter value |
| 127 | ERROR_PROC_NOT_FOUND | Function doesn't exist in DLL |
| 998 | ERROR_NOACCESS | Invalid pointer (buffer issue) |
| 1004 | ERROR_INVALID_FLAGS | Invalid flag combination |
| 1400 | ERROR_INVALID_WINDOW_HANDLE | Invalid window handle |
| 1413 | ERROR_INVALID_MONITOR_HANDLE | Invalid monitor handle |

**Error Handling Pattern:**
```ahk
result := DllCall("SetWindowPos", "Ptr", hwnd, ...)
if (!result) {
    errorCode := A_LastError
    switch errorCode {
        case 5: MsgBox "Access denied"
        case 1400: MsgBox "Invalid window handle"
        default: MsgBox "Error code: " errorCode
    }
}
```

**Checking A_LastError:**
- Some functions return 0 on success (e.g., GetLastError itself)
- Most Windows APIs return 0/NULL on failure and set A_LastError
- Always check function-specific documentation for return value meaning
- A_LastError = 0 usually means success, but not always

## Type Decision Matrix

Quick lookup: Which AHK type for which Windows type?

### Handle Types → Ptr

| Windows Type | AHK Type | Example |
|--------------|----------|---------|
| HWND | Ptr | Window handle |
| HANDLE | Ptr | Generic handle |
| HDC | Ptr | Device context |
| HBITMAP | Ptr | Bitmap handle |
| HBRUSH | Ptr | Brush handle |
| HPEN | Ptr | Pen handle |
| HFONT | Ptr | Font handle |
| HICON | Ptr | Icon handle |
| HCURSOR | Ptr | Cursor handle |
| HMENU | Ptr | Menu handle |
| HINSTANCE | Ptr | Instance handle |
| HMODULE | Ptr | Module handle |
| HKEY | Ptr | Registry key handle |

### Integer Types by Size

| Size | Signed | Unsigned | Example |
|------|--------|----------|---------|
| 1 byte | Char | UChar | BYTE |
| 2 bytes | Short | UShort | WORD |
| 4 bytes | Int | UInt | LONG, DWORD |
| 8 bytes | Int64 | UInt64 | LONGLONG |
| Pointer-sized | Ptr | UPtr | SIZE_T |

### Pointer Types → Ptr

| Windows Type | AHK Type | Notes |
|--------------|----------|-------|
| LPVOID | Ptr | Generic pointer |
| PVOID | Ptr | Generic pointer |
| LPCVOID | Ptr | Const pointer |
| Any LP* type | Ptr | Long pointer |
| Any P* type | Ptr | Pointer |

### String Types

| Windows Type | Input | Output |
|--------------|-------|--------|
| LPSTR | "AStr" | "Ptr" + Buffer |
| LPCSTR | "AStr" | N/A (input only) |
| LPWSTR | "WStr" or "Str" | "Ptr" + Buffer |
| LPCWSTR | "WStr" or "Str" | N/A (input only) |
| LPTSTR | "Str" | "Ptr" + Buffer |
| LPCTSTR | "Str" | N/A (input only) |

### Boolean and Status Types

| Windows Type | AHK Type | Check |
|--------------|----------|-------|
| BOOL | Int | if (result) |
| BOOLEAN | Int | if (result) |
| HRESULT | Int | if (result >= 0) |
| NTSTATUS | UInt | Check specific code |

## Common Gotchas Checklist

Quick reference of mistakes to avoid.

### Critical Errors (Cause Crashes)

- [ ] **Unquoted types in v2** → Quote all type strings: `"UInt"`, `"Ptr"`
- [ ] **Int for handles on 64-bit** → Use `Ptr` for ALL handles/pointers
- [ ] **No buffer allocation** → Use `Buffer(size, 0)` before passing to output params
- [ ] **Wrong buffer size** → Calculate: `chars × 2` for UTF-16 strings
- [ ] **Type mismatch** → Match AHK type to C type exactly
- [ ] **Missing VarRef & in v2** → Use `&var` for output parameters

### Common Mistakes (Wrong Results)

- [ ] **Wrong case function name** → Match exact export name case
- [ ] **Missing return type for pointers** → Specify `"Ptr"` as return type
- [ ] **Wrong parameter count** → Count and match MSDN signature
- [ ] **A_LastError not checked** → Check after every API call
- [ ] **Struct size not set** → NumPut size in cbSize field
- [ ] **Forgot StrGet after buffer fill** → Use `StrGet(buffer, encoding)`

### Platform Issues

- [ ] **32-bit vs 64-bit struct sizes** → Use `A_PtrSize` for dynamic sizing
- [ ] **Cdecl calling convention** → Specify `"Cdecl"` on 32-bit if needed
- [ ] **Pointer truncation** → Always use Ptr, never Int for addresses
- [ ] **Testing on single platform** → Test on both 32-bit and 64-bit

### Memory Management

- [ ] **Not cleaning up handles** → Call `CloseHandle()` on handles
- [ ] **Not freeing libraries** → Call `FreeLibrary()` on LoadLibrary results
- [ ] **Buffer too small** → Always allocate sufficient size
- [ ] **Zero-initialization** → Use second param in Buffer: `Buffer(size, 0)`

### Error Handling

- [ ] **Not checking return value** → Always check if function succeeded
- [ ] **Not checking A_LastError** → Check for error codes on failure
- [ ] **Not checking ErrorLevel** → Check for DllCall-level errors
- [ ] **No Try-Catch in development** → Catch exceptions during testing

## Debugging Decision Tree

Systematic debugging process.

### Step 1: Check ErrorLevel

```ahk
result := DllCall("FunctionName", ...)
MsgBox "ErrorLevel: " ErrorLevel
```

| ErrorLevel | Action |
|------------|--------|
| -4 | Check function name spelling and case |
| -3 | Verify DLL path and bitness matches AHK |
| -2 | Check type name spelling, add quotes in v2 |
| A-n | Count parameters against MSDN |
| Blank | Check memory access, validate all pointers |

### Step 2: Check Return Value and A_LastError

```ahk
result := DllCall("FunctionName", ...)
if (!result) {
    MsgBox "Failed. Error: " A_LastError
}
```

| A_LastError | Likely Cause |
|-------------|--------------|
| 5 | Permission issue, try running as admin |
| 6 | Handle is invalid or already closed |
| 87 | Parameter value is wrong/out of range |
| 127 | Function doesn't exist (wrong DLL) |
| 998 | Buffer pointer issue (size mismatch) |
| 1400 | Window handle is invalid |

### Step 3: Verify Parameter Types

Go through each parameter:
1. Find parameter in MSDN signature
2. Look up Windows type in translation table
3. Verify AHK type matches
4. Check for Ptr vs Int confusion
5. Verify string type (input vs output)

### Step 4: Test Minimal Example

```ahk
; This should always work:
DllCall("MessageBox", "Ptr", 0, "Str", "Test", "Str", "Title", "UInt", 0)
```

If minimal example works → problem is in your specific call
If minimal example fails → AHK installation issue

### Step 5: Validate All Pointers

```ahk
; Check handle validity
if (!hwnd || hwnd == -1) {
    MsgBox "Invalid handle"
    return
}

; Check buffer allocation
buffer := Buffer(size, 0)
MsgBox "Buffer size: " buffer.Size
```

## Quick Patterns Library

Common DllCall patterns for copy-paste reference.

### Pattern: Get Cursor Position

```ahk
pt := Buffer(8, 0)
DllCall("GetCursorPos", "Ptr", pt)
x := NumGet(pt, 0, "Int")
y := NumGet(pt, 4, "Int")
```

### Pattern: Get Window Rect

```ahk
rect := Buffer(16, 0)
DllCall("GetWindowRect", "Ptr", hwnd, "Ptr", rect)
left := NumGet(rect, 0, "Int")
top := NumGet(rect, 4, "Int")
right := NumGet(rect, 8, "Int")
bottom := NumGet(rect, 12, "Int")
```

### Pattern: Get Window Text

```ahk
length := DllCall("GetWindowTextLength", "Ptr", hwnd, "Int") + 1
buffer := Buffer(length * 2, 0)
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", length)
title := StrGet(buffer, "UTF-16")
```

### Pattern: Output Parameter

```ahk
processId := 0
threadId := DllCall("GetWindowThreadProcessId", "Ptr", hwnd, "UInt*", &processId, "UInt")
```

### Pattern: Two-Call Size Query

```ahk
; First call: get required size
size := 0
DllCall("SomeFunction", "Ptr", 0, "Int*", &size)

; Second call: get data
buffer := Buffer(size, 0)
DllCall("SomeFunction", "Ptr", buffer, "Int*", &size)
```

### Pattern: Struct with cbSize

```ahk
structSize := 40
info := Buffer(structSize, 0)
NumPut("UInt", structSize, info, 0)  ; Set cbSize
DllCall("GetMonitorInfo", "Ptr", hMonitor, "Ptr", info)
```

### Pattern: Error Handling

```ahk
result := DllCall("FunctionName", ...)
if (!result) {
    errorCode := A_LastError
    MsgBox "Failed with error: " errorCode
    return false
}
return true
```

### Pattern: Handle Cleanup

```ahk
hFile := DllCall("CreateFile", ..., "Ptr")
if (!hFile || hFile == -1) {
    return false
}

; Use file...

DllCall("CloseHandle", "Ptr", hFile)
```

### Pattern: Flag Combination

```ahk
; Combine flags with bitwise OR
flags := 0x0001 | 0x0002 | 0x0040  ; NOSIZE|NOMOVE|SHOWWINDOW
DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0, "Int", 0, "Int", 0,
        "Int", 0, "Int", 0, "UInt", flags)
```

### Pattern: Platform-Specific Struct

```ahk
; Adjust size based on pointer size
structSize := 20 + A_PtrSize
buffer := Buffer(structSize, 0)

offset := 0
NumPut("UInt", value1, buffer, offset)
offset += 4
NumPut("Ptr", ptrValue, buffer, offset)
offset += A_PtrSize
NumPut("UInt", value2, buffer, offset)
```

---

**Related Documentation:**
- See docs/manuals/ahk2/explanation/dllcall-memory-model.md for conceptual understanding
- See docs/manuals/ahk2/how-to/dllcall-common-tasks.md for task-specific guides
- See docs/manuals/ahk2/tutorials/dllcall-basics.md for learning path
