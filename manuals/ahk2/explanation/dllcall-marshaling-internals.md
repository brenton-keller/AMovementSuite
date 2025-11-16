# DllCall Marshaling Internals

## What This Document Explains

This document answers **why** DllCall works the way it does at the memory and CPU level. When you write `DllCall("SetWindowPos", "Ptr", hwnd, ...)`, what actually happens between AutoHotkey's high-level values and the low-level binary interface that Windows API functions expect? Why do incorrect type specifications cause crashes? Why does 64-bit behave differently from 32-bit?

Understanding these internals transforms you from someone who copies DllCall patterns to someone who can confidently design new API calls, debug mysterious failures, and understand performance implications.

## Core Concept: The Binary Translation Problem

AutoHotkey stores data in its own internal formats. A variable containing the number 42 exists as an AHK object with type metadata, while Windows API functions expect raw binary integers at specific memory locations. **DllCall is the bridge that translates AHK's high-level representation into the exact binary layout that C functions expect.**

This translation must be perfect. A single byte misalignment or type misinterpretation causes catastrophic failures—not graceful errors, but immediate crashes that terminate your script.

## Memory Layout Before the Call

### Parameter Storage in AutoHotkey

When you write:
```ahk
result := DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0, "Int", 100, "Int", 200,
                  "Int", 300, "Int", 400, "UInt", 0x0405)
```

AutoHotkey internally holds these values:
- `hwnd`: An integer value (say, 197402) stored in an AHK variable object
- `0`: A literal integer constant
- `100, 200, 300, 400`: Integer literals
- `0x0405`: A literal integer constant (1029 in decimal)

**None of these are in C-compatible format yet.** AHK variables carry type information, reference counts, and other metadata that C functions don't understand. The type strings ("Ptr", "Int", "UInt") tell DllCall how to perform the conversion.

### The Translation Table

Each type specification triggers specific binary formatting:

```
Type String    →    Binary Format                    Size
═══════════════════════════════════════════════════════════
"Ptr"          →    Raw pointer/address value       8 bytes (64-bit)
                                                     4 bytes (32-bit)

"Int"          →    32-bit signed integer           4 bytes
                    Two's complement format

"UInt"         →    32-bit unsigned integer         4 bytes
                    Binary format

"Int64"        →    64-bit signed integer           8 bytes
                    Two's complement format

"Str"          →    Pointer to UTF-16 string        8/4 bytes
                    String must be null-terminated

"Float"        →    IEEE 754 single precision       4 bytes

"Double"       →    IEEE 754 double precision       8 bytes
```

**Critical insight:** When you specify "Ptr" for the hwnd parameter, DllCall extracts the numeric value from the AHK variable and formats it as a raw 64-bit (or 32-bit) binary value. If you incorrectly specified "Int", DllCall would only use 32 bits, truncating the upper half on 64-bit systems.

## The Calling Convention: How Parameters Reach the Function

### 64-bit Calling Convention (x64 Windows)

On 64-bit Windows, the calling convention follows Microsoft's x64 ABI (Application Binary Interface):

**First four parameters** go into CPU registers:
```
Parameter 1  →  RCX register (64-bit general purpose)
Parameter 2  →  RDX register (64-bit general purpose)
Parameter 3  →  R8  register (64-bit general purpose)
Parameter 4  →  R9  register (64-bit general purpose)
```

**Fifth and subsequent parameters** go onto the call stack in left-to-right order.

**Additionally:**
- Caller must allocate 32 bytes of "shadow space" on stack for the callee to spill registers if needed
- Stack must be 16-byte aligned before the CALL instruction
- Return value comes back in RAX register (or XMM0 for floats)

### Visual Representation of SetWindowPos Call (64-bit)

```
Memory State Before Call:
═══════════════════════════════════════════════════════════════

CPU Registers:
┌─────────────────────────────────────────────────────────────┐
│ RCX  = 0x0000000000030C1A    ← Parameter 1: hwnd           │
│ RDX  = 0x0000000000000000    ← Parameter 2: hWndInsertAfter│
│ R8   = 0x0000000000000064    ← Parameter 3: X (100)        │
│ R9   = 0x00000000000000C8    ← Parameter 4: Y (200)        │
└─────────────────────────────────────────────────────────────┘

Call Stack (growing downward):
┌──────────────────────────────────┐  ← RSP (stack pointer)
│ [Shadow space - 32 bytes]        │
│  (reserved for callee)           │
├──────────────────────────────────┤
│ 0x0000000000000405               │  ← Parameter 7: flags
├──────────────────────────────────┤
│ 0x0000000000000190               │  ← Parameter 6: cy (400)
├──────────────────────────────────┤
│ 0x000000000000012C               │  ← Parameter 5: cx (300)
├──────────────────────────────────┤
│ [Return address]                 │
└──────────────────────────────────┘
```

**This is why type specifications matter:** If you specify "Int" instead of "Ptr" for parameter 1, only the lower 32 bits get loaded into RCX. On 64-bit systems, this effectively zeros out the upper half of the handle, creating an invalid pointer.

### 32-bit Calling Convention (stdcall)

On 32-bit Windows, most API functions use the "stdcall" convention:

**All parameters** go onto the stack in **right-to-left** order:
```
Last parameter pushed first
...
First parameter pushed last
Then CALL instruction
```

**Stack layout before call:**
```
Stack (32-bit stdcall):
┌──────────────────────────────┐  ← ESP before CALL
│ 0x00000405                   │  ← Parameter 7: flags
├──────────────────────────────┤
│ 0x00000190                   │  ← Parameter 6: cy (400)
├──────────────────────────────┤
│ 0x0000012C                   │  ← Parameter 5: cx (300)
├──────────────────────────────┤
│ 0x000000C8                   │  ← Parameter 4: Y (200)
├──────────────────────────────┤
│ 0x00000064                   │  ← Parameter 3: X (100)
├──────────────────────────────┤
│ 0x00000000                   │  ← Parameter 2: hWndInsertAfter
├──────────────────────────────┤
│ 0x00030C1A                   │  ← Parameter 1: hwnd (4 bytes!)
└──────────────────────────────┘
```

**Key differences:**
- Stack-based instead of register-based
- Parameters pushed right-to-left
- Callee cleans up the stack (pops parameters before returning)
- Return value in EAX register
- Handles are only 4 bytes (pointers are 32-bit)

This is why "Ptr" types automatically adjust size: on 32-bit AHK, Ptr = 4 bytes; on 64-bit AHK, Ptr = 8 bytes.

## The Marshaling Process: Step by Step

### Phase 1: Type Validation and Parsing

When DllCall encounters your call, it first parses the parameter list:

1. **Extract function name:** "SetWindowPos"
2. **Parse parameter pairs:** ("Ptr", hwnd), ("Ptr", 0), ("Int", 100), etc.
3. **Validate type strings:** Ensure "Ptr", "Int", "UInt" are recognized types
4. **Count parameters:** 7 parameters detected
5. **Determine return type:** Default "Int" (none specified)

If any type string is invalid or misspelled, DllCall throws an error immediately. This is **compile-time safety** in action.

### Phase 2: Function Resolution

DllCall must locate the actual function code in memory:

```
Resolution Process:
┌──────────────────────────────────────────────────┐
│ Function name: "SetWindowPos"                    │
│                                                  │
│ Step 1: Is DLL specified?                        │
│         No → Search system DLLs                  │
│                                                  │
│ Step 2: Try user32.dll (common for window APIs) │
│         LoadLibrary("user32.dll") if needed      │
│         → Returns HMODULE (base address)         │
│                                                  │
│ Step 3: GetProcAddress(hModule, "SetWindowPos") │
│         → Returns function pointer: 0x7FF8...    │
│                                                  │
│ Step 4: Cache for future calls (optimization)   │
└──────────────────────────────────────────────────┘
```

**Performance note:** The first call to a DllCall with a specific function name incurs DLL loading and symbol resolution overhead. Subsequent calls use the cached function pointer, making them much faster.

### Phase 3: Parameter Marshaling

Now DllCall converts each AHK value to binary format:

**For "Ptr" type (hwnd = 197402):**
```
AHK Variable:
┌────────────────────────────────┐
│ Type: Integer                  │
│ Value: 197402                  │
│ RefCount: 1                    │
│ [Other metadata...]            │
└────────────────────────────────┘
         ↓ Extract numeric value
         ↓ Format as platform-sized pointer
64-bit Binary:
┌────────────────────────────────┐
│ 0x00 00 00 00 00 03 0C 1A      │  ← 8 bytes
└────────────────────────────────┘
```

**For "Int" type (X = 100):**
```
AHK Literal: 100
         ↓ Format as signed 32-bit integer
Binary:
┌────────────────┐
│ 0x00 00 00 64  │  ← 4 bytes, little-endian
└────────────────┘
```

**For "Str" type (if we had a string parameter):**
```
AHK String: "Hello"
         ↓ Get pointer to internal UTF-16 buffer
         ↓ String already stored as UTF-16 in AHK v2
Binary (what gets passed):
┌────────────────────────────────┐
│ 0x00 00 01 A2 34 56 78 90      │  ← Pointer address
└────────────────────────────────┘
         ↓ Points to:
┌────────────────────────────────┐
│ 'H' 'e' 'l' 'l' 'o' '\0'       │  ← UTF-16: each char = 2 bytes
│ 48 00 65 00 6C 00 6C 00 6F 00  │
│ 00 00                          │
└────────────────────────────────┘
```

### Phase 4: Stack and Register Setup

On **64-bit systems:**

```assembly
; Pseudo-assembly showing what DllCall generates internally:

mov   rcx, 0x30C1A          ; Parameter 1 → RCX
mov   rdx, 0x0              ; Parameter 2 → RDX
mov   r8d, 0x64             ; Parameter 3 → R8 (32-bit move for Int)
mov   r9d, 0xC8             ; Parameter 4 → R9 (32-bit move for Int)

; Allocate stack space and align
sub   rsp, 0x38             ; 32 bytes shadow + 24 bytes params + alignment

; Push parameters 5-7 onto stack
mov   dword ptr [rsp+0x20], 0x12C   ; Parameter 5: cx
mov   dword ptr [rsp+0x28], 0x190   ; Parameter 6: cy
mov   dword ptr [rsp+0x30], 0x405   ; Parameter 7: flags

; Make the call
call  qword ptr [SetWindowPos_addr]

; After return, extract result from RAX
mov   [result_storage], eax

; Clean up stack
add   rsp, 0x38
```

On **32-bit systems:**

```assembly
; Pseudo-assembly for 32-bit stdcall:

; Push parameters in reverse order (right to left)
push  0x405                 ; Parameter 7
push  0x190                 ; Parameter 6
push  0x12C                 ; Parameter 5
push  0xC8                  ; Parameter 4
push  0x64                  ; Parameter 3
push  0x0                   ; Parameter 2
push  0x30C1A               ; Parameter 1

; Call function (callee cleans stack in stdcall)
call  dword ptr [SetWindowPos_addr]

; Extract result from EAX
mov   [result_storage], eax

; Stack automatically cleaned by callee
```

### Phase 5: The Actual Function Call

At this point, control transfers to the Windows API function. From the CPU's perspective:

```
Before CALL:
┌───────────────────────────────────┐
│ RIP (instruction pointer)         │
│ = Address in AutoHotkey code      │
└───────────────────────────────────┘

CALL instruction executes:
┌───────────────────────────────────┐
│ 1. Push return address to stack   │
│ 2. Jump to function address       │
└───────────────────────────────────┘

During function execution:
┌───────────────────────────────────┐
│ RIP = Address in user32.dll       │
│ Function reads parameters from    │
│ registers/stack as needed         │
│ Function performs its work        │
│ Function writes return value to   │
│ RAX (or EAX on 32-bit)            │
└───────────────────────────────────┘

After RET instruction:
┌───────────────────────────────────┐
│ RIP = Return address (back to AHK)│
│ RAX contains return value         │
└───────────────────────────────────┘
```

**This is the critical moment:** The Windows API function is now executing with parameters in the exact binary layout it expects. If DllCall marshaled incorrectly, the function reads garbage data, leading to crashes or wrong behavior.

## Phase 6: Return Value Extraction and Error Handling

After the function returns, DllCall must translate the binary return value back to an AHK value:

### Return Value Processing

```
Function returns with RAX = 0x0000000000000001 (success)

DllCall examines return type specification:
  Default: "Int" (since not specified)

Extract 32-bit value from RAX:
┌────────────────┐
│ RAX = 0x...001 │
│        └──┬──┘ │
│           ↓    │
│       EAX = 1  │  ← Lower 32 bits
└────────────────┘

Convert to AHK Integer:
┌────────────────────────┐
│ Create AHK variable    │
│ Type: Integer          │
│ Value: 1               │
└────────────────────────┘
```

**Critical:** If the function returns a pointer but you didn't specify "Ptr" as return type, DllCall only extracts the lower 32 bits on 64-bit systems:

```
LoadLibrary returns: RAX = 0x00007FF8A2B40000 (DLL base address)

With default "Int" return type:
  Extract EAX = 0xA2B40000  ← Upper bits lost!
  Handle is now truncated and invalid

With "Ptr" return type:
  Extract full RAX = 0x00007FF8A2B40000
  Handle is valid
```

### A_LastError Mechanism

Immediately after extracting the return value, DllCall calls `GetLastError()`:

```
Timeline of Error Capture:
═══════════════════════════════════════════════════════════

1. SetWindowPos executes
   └→ If error occurs, Windows sets thread-local error code

2. SetWindowPos returns
   └→ Control back to DllCall

3. DllCall immediately calls GetLastError()
   └→ Captures error code before it's overwritten

4. DllCall stores error code in A_LastError
   └→ Now accessible to your script

5. DllCall returns to your AHK code
   └→ You can check both return value and A_LastError
```

**Why immediate capture matters:** Many Windows API functions (even successful ones) might call other APIs that set error codes. If DllCall didn't capture GetLastError() immediately, the error code could be overwritten by subsequent operations.

**Thread-local storage:** Each thread has its own error code. GetLastError() retrieves the code for the current thread only. This is why multi-threaded scripts must be careful—error codes don't cross thread boundaries.

### Complete Round-Trip Visualization

```
┌─────────────────────────────────────────────────────────────┐
│                    AutoHotkey Script                        │
│  result := DllCall("SetWindowPos", "Ptr", hwnd, ...)        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                  DllCall Marshaling Engine                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Parse types: "Ptr", "Ptr", "Int"...               │  │
│  │ 2. Validate: All types recognized ✓                  │  │
│  │ 3. Resolve: Find SetWindowPos in user32.dll          │  │
│  │ 4. Convert: AHK values → Binary format               │  │
│  │ 5. Setup: Load registers/stack per calling convention│  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓ CALL instruction
┌─────────────────────────────────────────────────────────────┐
│                  Windows API Function                       │
│               (SetWindowPos in user32.dll)                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Read parameters from RCX, RDX, R8, R9, stack      │  │
│  │ 2. Validate window handle                            │  │
│  │ 3. Perform window positioning                        │  │
│  │ 4. Set error code if failure (SetLastError)          │  │
│  │ 5. Write result to RAX (1 = success, 0 = failure)   │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓ RET instruction
┌─────────────────────────────────────────────────────────────┐
│                  DllCall Return Processing                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Extract return value from RAX                     │  │
│  │ 2. Call GetLastError() → A_LastError                 │  │
│  │ 3. Convert binary return to AHK Integer              │  │
│  │ 4. Clean up stack (if needed)                        │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                    AutoHotkey Script                        │
│  result now contains 1 (success)                            │
│  A_LastError contains 0 (no error)                          │
└─────────────────────────────────────────────────────────────┘
```

## Why Incorrect Types Cause Crashes

### Memory Misinterpretation

When you tell DllCall a parameter is type "Str" but pass an integer:

```ahk
; WRONG - will crash
hwnd := 197402
DllCall("SetWindowPos", "Str", hwnd, ...)  ; Disaster!
```

**What happens:**

```
DllCall sees "Str" type:
  → Interprets hwnd value as a memory address
  → Tries to read string data from address 0x30C1A

Memory at 0x30C1A:
┌─────────────────────────────────────┐
│ ??? (probably unmapped memory)      │
│ OR garbage data                     │
│ OR protected system memory          │
└─────────────────────────────────────┘

Result: ACCESS VIOLATION
  → Windows terminates the process
  → Script crashes immediately
```

The CPU attempts to read from an invalid memory address, triggering a hardware exception. The operating system cannot allow this and kills the process.

### Size Mismatches

Using "Int" instead of "Ptr" for handles:

```ahk
; Works on 32-bit, FAILS on 64-bit
DllCall("SetWindowPos", "Int", hwnd, ...)
```

**On 64-bit:**

```
Actual handle value: 0x00007FF8A2B40000 (8 bytes needed)

DllCall marshals as "Int":
  → Only uses lower 32 bits: 0xA2B40000
  → Passes this to RCX

SetWindowPos receives:
  RCX = 0x00000000A2B40000  ← Invalid handle!

SetWindowPos tries to use this handle:
  → Looks up handle in kernel handle table
  → Handle 0xA2B40000 doesn't exist
  → Returns failure, sets ERROR_INVALID_HANDLE

OR worse:
  → Handle 0xA2B40000 exists but points to different object
  → SetWindowPos operates on wrong window
  → Mysterious behavior ensues
```

### Buffer Underallocation

Telling a function the buffer is larger than allocated:

```ahk
; WRONG - buffer overflow
buffer := Buffer(100, 0)
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buffer, "Int", 500)
                                                            └─ Claims 500 chars!
```

**What happens:**

```
GetWindowText receives:
  Parameter 2: Pointer to buffer (valid)
  Parameter 3: Max chars = 500

GetWindowText writes window title:
  Writes up to 500 characters (1000 bytes for UTF-16)

Buffer layout:
┌────────────────────────────────┐
│ buffer[0..99]   ← Allocated    │  100 bytes
├────────────────────────────────┤
│ buffer[100..199] ← NOT YOURS!  │  \
│ buffer[200..299] ← NOT YOURS!  │   } Writing to unowned memory
│ ...                            │  /
│ buffer[900..999] ← NOT YOURS!  │
└────────────────────────────────┘

Result:
  → Heap corruption (overwrites other allocations)
  → Crashes later (not immediately - harder to debug!)
  → Potential security vulnerability
```

### The Alignment Problem

Structs with incorrect padding:

```ahk
; Suppose a struct is:
; typedef struct {
;     DWORD field1;    // 4 bytes at offset 0
;     PVOID field2;    // 8 bytes at offset 8 (NOT 4!)
; } MYSTRUCT;

; WRONG - no padding
mystruct := Buffer(12, 0)  ; 4 + 8 = 12
NumPut("UInt", 100, mystruct, 0)     ; Offset 0 ✓
NumPut("Ptr", handle, mystruct, 4)   ; Offset 4 ✗ Should be 8!
```

**Memory layout:**

```
What you created:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ F1 │ F1 │ F1 │ F1 │ F2 │ F2 │ F2 │ F2 │ F2 │ F2 │ F2 │ F2 │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  0    1    2    3    4    5    6    7    8    9   10   11

What the function expects:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ F1 │ F1 │ F1 │ F1 │ -- │ -- │ -- │ -- │ F2 │ F2 │ F2 │ F2 │ F2 │ F2 │ F2 │ F2 │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  0    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15
                      └─ Padding ─┘

Function reads field2 at offset 8:
  → Gets bytes 8-15 from your buffer
  → Your buffer only has 12 bytes
  → Reads beyond allocation
  → Crash or garbage data
```

**Why padding exists:** CPU performance. Most CPUs access aligned data faster. A 64-bit value at an 8-byte boundary can be loaded in one instruction; misaligned requires multiple loads and shifts.

## Performance Implications

### Cost of Each DllCall

**First call to a function:**
1. Parse type strings: ~few microseconds
2. Resolve function address: ~tens of microseconds (DLL load + symbol lookup)
3. Marshal parameters: ~microseconds per parameter
4. Execute call: ~depends on function complexity
5. Capture error: ~1-2 microseconds (GetLastError call)
6. Unmarshal return: ~few microseconds
7. Cache function pointer: ~microseconds

**Total first call:** ~50-200 microseconds

**Subsequent calls:**
1. Use cached function pointer: ~microseconds
2. Marshal parameters: ~microseconds per parameter
3. Execute call: ~depends on function
4. Capture error: ~1-2 microseconds
5. Unmarshal return: ~few microseconds

**Total subsequent calls:** ~10-50 microseconds + function execution time

### Optimization Strategies

**Cache DLL handles:**
```ahk
; Instead of:
Loop 1000 {
    DllCall("MyDll.dll\MyFunc")  ; Loads DLL 1000 times
}

; Do this:
hDll := DllCall("LoadLibrary", "Str", "MyDll.dll", "Ptr")
Loop 1000 {
    DllCall(hDll . "\MyFunc")  ; Reuses loaded DLL
}
DllCall("FreeLibrary", "Ptr", hDll)
```

**Pre-allocate buffers:**
```ahk
; Instead of:
Loop 1000 {
    buf := Buffer(16, 0)  ; Allocates 1000 times
    DllCall("GetCursorPos", "Ptr", buf)
}

; Do this:
buf := Buffer(16, 0)  ; Allocate once
Loop 1000 {
    DllCall("GetCursorPos", "Ptr", buf)  ; Reuse buffer
}
```

**Avoid string conversions in loops:**
```ahk
; Instead of:
Loop Files, "*.txt" {
    DllCall("SomeFunc", "Str", A_LoopFileName)  ; May require conversion
}

; If function accepts wide strings:
Loop Files, "*.txt" {
    DllCall("SomeFunc", "WStr", A_LoopFileName)  ; Direct UTF-16 pass
}
```

## Platform Differences: 32-bit vs 64-bit Deep Dive

### Pointer Size Impact

**The most critical difference:**

```
32-bit AHK:
  Ptr type  = 4 bytes
  UPtr type = 4 bytes
  Maximum addressable memory = 4 GB (2^32)
  Handles fit in 32 bits

64-bit AHK:
  Ptr type  = 8 bytes
  UPtr type = 8 bytes
  Maximum addressable memory = 16 EB (2^64 theoretically, 256 TB practically)
  Handles require 64 bits
```

**Code portability:**

```ahk
; WRONG - breaks on 64-bit
GetWindowSize(hwnd) {
    rect := Buffer(16, 0)  ; Assumes 4 Ints
    DllCall("GetWindowRect", "Int", hwnd, "Ptr", rect)  ; Truncates handle!
    return NumGet(rect, 8, "Int") - NumGet(rect, 0, "Int")
}

; CORRECT - works on both
GetWindowSize(hwnd) {
    rect := Buffer(16, 0)  ; RECT is always 4 Ints (16 bytes)
    DllCall("GetWindowRect", "Ptr", hwnd, "Ptr", rect)  ; Ptr adjusts automatically
    return NumGet(rect, 8, "Int") - NumGet(rect, 0, "Int")
}
```

### Struct Size Variations

Some structs contain pointers, changing size based on platform:

```
FLASHWINFO struct:
32-bit version (20 bytes):
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ cbSize  │  hwnd   │ dwFlags │ uCount  │dwTimeout│
│ 4 bytes │ 4 bytes │ 4 bytes │ 4 bytes │ 4 bytes │
└─────────┴─────────┴─────────┴─────────┴─────────┘
  Offset:    0         4         8        12        16

64-bit version (24 bytes):
┌─────────┬────────────────┬─────────┬─────────┬─────────┐
│ cbSize  │ (pad) │  hwnd  │ dwFlags │ uCount  │dwTimeout│
│ 4 bytes │4 bytes│8 bytes │ 4 bytes │ 4 bytes │ 4 bytes │
└─────────┴───────┴────────┴─────────┴─────────┴─────────┘
  Offset:    0       4        8         16        20        24
                     └──HWND at offset 8!
```

**Portable struct handling:**

```ahk
FlashWindow(hwnd) {
    structSize := 20 + A_PtrSize - 4  ; 20 on 32-bit, 24 on 64-bit
    fwi := Buffer(structSize, 0)

    offset := 0
    NumPut("UInt", structSize, fwi, offset)
    offset += 4

    if (A_PtrSize = 8)  ; 64-bit padding
        offset += 4

    NumPut("Ptr", hwnd, fwi, offset)
    offset += A_PtrSize

    NumPut("UInt", 0x0F, fwi, offset)
    ; ... etc
}
```

### Register Usage Differences

**32-bit registers (stdcall):**
- EAX, EBX, ECX, EDX: General purpose (32-bit)
- Parameters on stack
- EAX for return value
- ESI, EDI preserved by callee

**64-bit registers (Microsoft x64):**
- RAX, RBX, RCX, RDX, R8-R15: General purpose (64-bit)
- First 4 params in RCX, RDX, R8, R9
- RAX for return value
- XMM0-XMM5 for float parameters
- RBX, RBP, RDI, RSI, RSP, R12-R15 preserved by callee

## Common Marshaling Errors and Their Memory Signatures

### Error 1: Null Pointer Dereference

**Code:**
```ahk
DllCall("SomeFunc", "Ptr", 0)  ; Passing NULL
; Function tries to read from address 0
```

**Memory access:**
```
Attempt to read from 0x0000000000000000:
  → Page fault exception
  → Windows: "Access violation reading location 0x00000000"
  → Script terminates
```

### Error 2: Stack Corruption

**Code:**
```ahk
; Function expects 5 params, you provide 4
DllCall("FiveParamFunc", "Int", 1, "Int", 2, "Int", 3, "Int", 4)
```

**What happens:**
```
Function expects:
┌─────┬─────┬─────┬─────┬─────┐
│ P1  │ P2  │ P3  │ P4  │ P5  │
└─────┴─────┴─────┴─────┴─────┘

You provided:
┌─────┬─────┬─────┬─────┐
│ P1  │ P2  │ P3  │ P4  │
└─────┴─────┴─────┴─────┘

Function reads P5 from garbage location:
  → Reads whatever happened to be on stack
  → Unpredictable behavior
  → May read return address as parameter!
  → Function may corrupt stack when cleaning up
```

### Error 3: Use After Free

**Code:**
```ahk
GetString() {
    buf := Buffer(100, 0)
    DllCall("SomeFunc", "Ptr", buf)
    return StrGet(buf)
}  ; buf is freed here

str := GetString()
; Later...
DllCall("ProcessString", "Ptr", buf)  ; buf is gone!
```

**Memory state:**
```
During GetString():
┌────────────────────┐
│ Buffer allocated   │  ← Valid memory
└────────────────────┘

After GetString() returns:
┌────────────────────┐
│ Memory freed       │  ← Now part of free list
│ May be reallocated │  ← Could contain anything
└────────────────────┘

DllCall tries to use freed memory:
  → Reads garbage data
  → OR crashes if memory was unmapped
  → OR corrupts new allocation
```

## Advanced: The Output Parameter Dance

Output parameters require special handling because data flows backward (function → caller):

### Simple Output Parameter

**C signature:**
```c
BOOL GetWindowThreadProcessId(
    HWND hWnd,
    LPDWORD lpdwProcessId  // Output: function writes here
);
```

**Memory flow:**

```
Before call:
┌──────────────────────────┐
│ Your variable: processId │
│ Value: 0 (uninitialized) │
│ Address: 0x00001234ABCD  │
└──────────────────────────┘

DllCall setup:
  Pass &processId (address 0x00001234ABCD) as parameter 2

During function execution:
┌──────────────────────────┐
│ Function receives        │
│ Pointer: 0x00001234ABCD  │
│                          │
│ Writes value 5678 to     │
│ *pointer (address)       │
└──────────────────────────┘

After call:
┌──────────────────────────┐
│ Your variable: processId │
│ Value: 5678 ✓           │
│ Address: 0x00001234ABCD  │
└──────────────────────────┘
```

**AHK code:**
```ahk
processId := 0
threadId := DllCall("GetWindowThreadProcessId", "Ptr", hwnd, "UInt*", &processId, "UInt")
; processId now contains the process ID written by the function
```

### Buffer Output Parameter

**C signature:**
```c
int GetWindowText(
    HWND hWnd,
    LPTSTR lpString,    // Output buffer
    int nMaxCount       // Buffer size in chars
);
```

**Memory flow:**

```
Before call:
┌────────────────────────────────────────┐
│ buffer := Buffer(520, 0)               │
│ ┌──────────────────────────────────┐   │
│ │ [520 bytes of zeros]             │   │
│ │ Address: 0x00002000DEAD          │   │
│ └──────────────────────────────────┘   │
└────────────────────────────────────────┘

DllCall setup:
  Parameter 2: Pointer to buffer (0x00002000DEAD)
  Parameter 3: 260 (max characters)

During function execution:
┌────────────────────────────────────────┐
│ GetWindowText writes to buffer:        │
│ Address 0x00002000DEAD:                │
│ ┌──────────────────────────────────┐   │
│ │ 'N' 'o' 't' 'e' 'p' 'a' 'd' '\0' │   │ (UTF-16)
│ │ 4E 00 6F 00 74 00 65 00 70 00... │   │
│ └──────────────────────────────────┘   │
└────────────────────────────────────────┘

After call:
  buffer contains the window text
  Use StrGet(buffer) to extract to AHK string
```

## Debunked Misconceptions

### Misconception 1: "Type specifications don't matter much"

**Wrong.** Types are **critical** for correct marshaling. Using "Int" instead of "Ptr" doesn't just risk failure—it **guarantees** failure on 64-bit systems with handle values exceeding 32 bits.

**Why it seems to work sometimes:** Small handle values (< 4 billion) fit in 32 bits. On test systems with few processes, handles might stay small, making "Int" appear to work. In production with many processes, handles grow larger and code breaks.

### Misconception 2: "32-bit and 64-bit work the same"

**Wrong.** The calling convention is fundamentally different:
- 32-bit: Stack-based parameter passing
- 64-bit: Register-based for first 4 parameters

Additionally:
- Pointer sizes differ (4 vs 8 bytes)
- Struct layouts differ (padding changes)
- Available address space differs (4 GB vs terabytes)

**Why it seems the same:** DllCall abstracts these differences when you use correct types. Using "Ptr" works because DllCall adjusts the size automatically.

### Misconception 3: "Strings are just passed directly"

**Wrong.** String passing involves:

1. **AHK v2 stores strings as UTF-16 internally**
2. **"Str" type passes a pointer to this internal buffer**
3. **For "AStr", DllCall allocates temporary buffer, converts UTF-16 → ANSI, passes pointer**
4. **After call, temporary ANSI buffer is freed**

**For output strings:**
- Never use "Str"/"AStr"/"WStr" types
- Must use "Ptr" with pre-allocated Buffer
- Must use StrGet() to extract after call

**Why it seems simple:** When passing string literals or simple variables for input, DllCall handles conversion automatically. Problems arise with output strings or lifetime issues.

### Misconception 4: "Buffer allocation isn't critical"

**Wrong.** Buffer management is one of the most crash-prone aspects:

**Under-allocation:**
```ahk
buf := Buffer(10, 0)
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buf, "Int", 100)
                                                        └─ Claim 100 chars
                                                           But only allocated 10 bytes!
```
Result: Buffer overflow, heap corruption, crashes

**Over-claiming:**
```ahk
buf := Buffer(1000, 0)
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buf, "Int", 500)
```
Result: May work, but wastes memory. Not a crash risk (unlike under-allocation).

**No initialization:**
```ahk
buf := Buffer(100)  ; No zero-fill
DllCall("GetWindowText", "Ptr", hwnd, "Ptr", buf, "Int", 50)
title := StrGet(buf)  ; May include garbage after string
```
Result: Garbage data, wrong string extraction

**Always:**
- Allocate sufficient space
- Zero-fill: `Buffer(size, 0)`
- Match size parameter to actual allocation (in characters for string APIs!)

## Summary: The Marshaling Mental Model

When you write DllCall, visualize this process:

1. **Your AHK values exist in high-level format** with type metadata
2. **DllCall's job is binary translation** according to type specifications
3. **Parameters get loaded into CPU registers/stack** per calling convention
4. **Control transfers to native code** which reads parameters as raw binary
5. **Native function writes result to register/memory** and sets error codes
6. **DllCall translates binary result back to AHK format** and captures errors
7. **You receive AHK value and A_LastError** for error checking

**Type specifications are the blueprint** for this translation. Wrong types mean wrong translation, leading to crashes, corruption, or mysterious failures.

**Platform differences are handled by "Ptr"** type, which adjusts size automatically. Never use Int/UInt for pointers.

**Calling conventions determine parameter locations.** You don't need to know assembly, but understanding that 64-bit uses registers first (while 32-bit uses stack) explains why performance differs and why parameter order matters.

**Error handling requires checking both** return value AND A_LastError, because Windows APIs signal errors differently (some return 0, some return NULL, some return -1).

Now when you see `DllCall("SetWindowPos", "Ptr", hwnd, "Ptr", 0, ...)`, you understand the invisible machinery: type parsing, function resolution, binary marshaling, register/stack setup, native execution, return extraction, and error capture. This knowledge transforms DllCall from mysterious incantation to predictable, debuggable tool.
