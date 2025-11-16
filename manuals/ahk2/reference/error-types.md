# Reference: Error Types

Complete error hierarchy with all properties, when each error is thrown, and handling patterns.

## Related Documentation
- docs/manuals/ahk2/reference/type-system.md - TypeError and ValueError in type operations
- docs/manuals/ahk2/reference/class-system.md - Error handling in classes
- docs/manuals/ahk2/reference/property-system.md - PropertyError in property access

---

## Error Class Hierarchy

```
Error (base class)
│
├── MemoryError
│
├── OSError
│   └── TimeoutError
│
├── TargetError
│
├── TypeError
│
├── UnsetError
│   └── UnsetItemError
│
├── MemberError
│   ├── PropertyError
│   └── MethodError
│
└── ValueError
    ├── IndexError
    └── ZeroDivisionError
```

---

## Error Object Properties

All error objects inherit these properties from the base `Error` class:

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `Message` | String | Human-readable error description | "Type mismatch" |
| `What` | String | Function/method name that threw | "MyFunction" |
| `Extra` | String | Additional context (often the invalid value) | "abc" |
| `File` | String | Full path to script file | "C:\Scripts\test.ahk" |
| `Line` | Integer | Line number where error occurred | 42 |
| `Stack` | String | Complete call stack trace (~2047 char limit) | See stack format below |

### OSError/TimeoutError Additional Property

| Property | Type | Description |
|----------|------|-------------|
| `Number` | Integer | Windows error code | `5` (Access Denied) |

---

## Error Types Reference

### MemoryError

**When Thrown:** System is out of memory (extremely rare on modern systems).

**Common Causes:**
- Allocating massive buffers
- Extreme memory leak
- System-wide memory exhaustion

**Example:**
```ahk
try {
    buf := Buffer(10000000000)  ; 10 GB
} catch MemoryError as e {
    MsgBox "Out of memory: " . e.Message
}
```

**Handling:** Usually cannot recover - log error and exit gracefully.

---

### OSError

**When Thrown:** Windows API call fails with error code.

**Common Causes:**
- File operations on locked/missing/read-only files
- Registry access denied
- Network operations fail
- Pixel operations fail
- Permission issues

**Properties:**
- `Number` - Windows error code
- `Message` - Formatted as "(code) Description"

**Common Error Codes:**

| Code | Constant | Description | Context |
|------|----------|-------------|---------|
| 2 | ERROR_FILE_NOT_FOUND | File not found | FileRead, FileOpen |
| 3 | ERROR_PATH_NOT_FOUND | Path not found | FileRead, DirCreate |
| 5 | ERROR_ACCESS_DENIED | Access denied | File/registry operations |
| 32 | ERROR_SHARING_VIOLATION | File in use | FileCopy, FileDelete |
| 183 | ERROR_ALREADY_EXISTS | Already exists | DirCreate |

**Examples:**
```ahk
; File not found
try {
    content := FileRead("missing.txt")
} catch OSError as e {
    if (e.Number = 2)
        MsgBox "File not found: " . e.Extra
    else
        MsgBox "OS Error " . e.Number . ": " . e.Message
}

; Access denied
try {
    FileDelete("C:\Windows\System32\important.dll")
} catch OSError as e {
    if (e.Number = 5)
        MsgBox "Access denied. Run as administrator?"
}

; File in use
try {
    FileCopy("source.txt", "dest.txt")
} catch OSError as e {
    if (e.Number = 32)
        MsgBox "File is locked. Close other programs."
}
```

**Format Example:**
```ahk
OSError(5).Message  ; Returns "(5) Access is denied."
```

---

### TimeoutError

**Extends:** OSError

**When Thrown:** SendMessage() times out waiting for response.

**Properties:** Same as OSError (includes `Number`)

**Example:**
```ahk
try {
    result := SendMessage(0x0010, 0, 0, , "ahk_class Notepad")
} catch TimeoutError as e {
    MsgBox "Window did not respond in time"
}
```

**Default Timeout:** 5000ms (configurable via SendMode)

---

### TypeError

**When Thrown:** Wrong type of value used where specific type expected.

**Common Causes:**
- Math operation on non-numeric string
- Wrong parameter type to function
- Invalid `x is y` class check
- Empty string in arithmetic (v2 change)

**Property `Extra`:** Contains the offending value as string

**Examples:**
```ahk
; Math on non-numeric
try {
    result := "text" + 5
} catch TypeError as e {
    MsgBox "Type error: " . e.Message
    MsgBox "Invalid value: " . e.Extra  ; "text"
}

; Empty string in arithmetic (v2)
try {
    x := ""
    result := x * 10
} catch TypeError as e {
    MsgBox "Cannot perform math on empty string"
}

; Wrong parameter type
MyFunc(x) {
    if !(x is "Integer")
        throw TypeError("Expected integer", -1, x)
}

try {
    MyFunc("abc")
} catch TypeError as e {
    MsgBox "Wrong type: " . e.Extra
}
```

---

### ValueError

**When Thrown:** Correct type but invalid value.

**Common Causes:**
- Out-of-range parameter
- Invalid option string
- Negative number where positive expected
- Invalid format string

**Property `Extra`:** Contains the invalid value

**Examples:**
```ahk
; Out of range
SetVolume(level) {
    if (level < 0 || level > 100)
        throw ValueError("Volume must be 0-100", -1, level)
    SoundSetVolume(level)
}

try {
    SetVolume(150)
} catch ValueError as e {
    MsgBox "Invalid value: " . e.Extra  ; "150"
}

; Invalid option
try {
    Hotkey "F1", MyFunc, "InvalidOption"
} catch ValueError as e {
    MsgBox "Invalid hotkey option: " . e.Message
}
```

---

### IndexError

**Extends:** ValueError

**When Thrown:** Array/object index out of range.

**Examples:**
```ahk
arr := [1, 2, 3]

try {
    value := arr[5]  ; Index out of range
} catch IndexError as e {
    MsgBox "Index error: " . e.Message
    MsgBox "Invalid index: " . e.Extra
}

; Safe access pattern
value := arr.Has(5) ? arr[5] : "default"
```

---

### ZeroDivisionError

**Extends:** ValueError

**When Thrown:** Division or modulo by zero.

**Examples:**
```ahk
try {
    result := 10 / 0
} catch ZeroDivisionError as e {
    MsgBox "Cannot divide by zero"
}

try {
    remainder := 10 % 0
} catch ZeroDivisionError as e {
    MsgBox "Cannot modulo by zero"
}

; Safe division pattern
SafeDivide(a, b) {
    if (b = 0)
        return 0  ; or return unset, or throw custom error
    return a / b
}
```

---

### TargetError

**When Thrown:** Target not found (very common in AHK2 window operations).

**Common Causes:**
- Window doesn't exist
- Control not found
- Menu item doesn't exist
- Hotstring not found

**Examples:**
```ahk
; Window operations
try {
    WinActivate("ahk_exe nonexistent.exe")
} catch TargetError as e {
    MsgBox "Window not found"
}

try {
    title := WinGetTitle("A")  ; No active window
} catch TargetError as e {
    MsgBox "No active window"
}

; Control operations
try {
    ControlGetText("Edit1", "ahk_class Notepad")
} catch TargetError as e {
    MsgBox "Control not found"
}

; Safe window operation pattern
if WinExist("ahk_class Notepad")
    WinActivate
else
    Run "notepad.exe"
```

---

### UnsetError

**When Thrown:** Variable/property accessed before assignment.

**Common Causes:**
- Reading uninitialized variable
- Property never set
- Increment/decrement on unset variable

**Examples:**
```ahk
; Uninitialized variable
try {
    MsgBox undeclaredVar
} catch UnsetError as e {
    MsgBox "Variable not initialized: " . e.What
}

; Unset property
class MyClass {
    Method() {
        return this.neverSet  ; UnsetError
    }
}

; Increment unset variable
try {
    counter++  ; counter was never initialized
} catch UnsetError as e {
    MsgBox "Cannot increment unset variable"
}

; Safe access with isset
if IsSet(myVar)
    MsgBox myVar
else
    MsgBox "Variable not set"
```

---

### UnsetItemError

**Extends:** UnsetError

**When Thrown:** Map/Array item doesn't exist.

**Examples:**
```ahk
myMap := Map()

try {
    value := myMap["nonexistent"]
} catch UnsetItemError as e {
    MsgBox "Key not found: " . e.Extra
}

; Safe access pattern
value := myMap.Has("key") ? myMap["key"] : "default"
value := myMap.Get("key", "default")  ; Preferred
```

**Distinction from IndexError:** `UnsetItemError` is for missing keys in Map. `IndexError` is for out-of-range array indices.

---

### MemberError

**Base class for PropertyError and MethodError.** Rarely thrown directly.

---

### PropertyError

**Extends:** MemberError

**When Thrown:** Property doesn't exist or is read-only/write-only.

**Common Causes:**
- Accessing non-existent property
- Writing to read-only property
- Reading write-only property

**Examples:**
```ahk
class Person {
    Age {
        get => this._age
        set => this._age := value
    }

    ID {
        get => this._id
        set => throw PropertyError("ID is read-only")
    }
}

person := Person()

try {
    person.ID := 123
} catch PropertyError as e {
    MsgBox "Cannot set: " . e.Message
}

try {
    MsgBox person.NonExistent
} catch PropertyError as e {
    MsgBox "Property doesn't exist: " . e.What
}
```

---

### MethodError

**Extends:** MemberError

**When Thrown:** Method doesn't exist.

**Examples:**
```ahk
obj := {}

try {
    obj.NonExistentMethod()
} catch MethodError as e {
    MsgBox "Method not found: " . e.What
}
```

---

## Creating Custom Errors

### Basic Custom Error

```ahk
class ValidationError extends ValueError {
    __New(message, fieldName := "", value := "") {
        super.__New(message, -1, value)
        this.FieldName := fieldName
    }
}

; Usage
ValidateAge(age) {
    if (age < 0)
        throw ValidationError("Age cannot be negative", "Age", age)
}

try {
    ValidateAge(-5)
} catch ValidationError as e {
    MsgBox "Validation failed: " . e.FieldName . " = " . e.Extra
}
```

### Complex Custom Error

```ahk
class HTTPError extends Error {
    __New(statusCode, message := "", url := "", responseBody := "") {
        super.__New(message || "HTTP Error", -1)
        this.StatusCode := statusCode
        this.URL := url
        this.ResponseBody := responseBody
    }

    ToString() {
        return Format("HTTP {1} at {2}: {3}",
                     this.StatusCode, this.URL, this.Message)
    }
}

; Usage
try {
    if (response.status = 404)
        throw HTTPError(404, "Not found", url, response.body)
} catch HTTPError as e {
    MsgBox e.ToString()
    FileAppend e.ResponseBody, "error_" . A_Now . ".html"
}
```

---

## Catching Specific Error Types

### Single Type

```ahk
try {
    FileRead("missing.txt")
} catch OSError as e {
    MsgBox "OS Error: " . e.Number
}
```

### Multiple Types (Hierarchical)

```ahk
try {
    obj.Property
} catch MemberError as e {
    ; Catches BOTH PropertyError and MethodError
    MsgBox "Member doesn't exist: " . e.What
} catch OSError as e {
    MsgBox "OS Error: " . e.Number
} catch as e {
    ; Catch everything else
    MsgBox "Other error: " . e.Message
}
```

### Specific Type Check

```ahk
try {
    RiskyOperation()
} catch as e {
    if (e is OSError)
        MsgBox "OS Error: " . e.Number
    else if (e is ValueError)
        MsgBox "Value Error: " . e.Extra
    else
        throw  ; Re-throw unhandled errors
}
```

---

## Call Stack Format

The `Stack` property contains a multi-line string with this format:

```
> MyFunction(param1,param2)
    MyInnerFunction()
        Line 42: result := operation()
```

**Format:**
- `>` indicates entry point
- Function names with parameters
- Indentation shows call depth
- Line numbers for specific operations

**Example:**
```ahk
try {
    Level1()
} catch as e {
    MsgBox e.Stack
}

Level1() => Level2()
Level2() => Level3()
Level3() => throw Error("Test")

; Stack output:
; > Level1()
;     Level2()
;         Level3()
```

**Limitation:** Stack string is truncated at ~2047 characters.

---

## Error Handling Patterns

### Type-Specific Recovery

```ahk
try {
    data := FileRead(path)
} catch OSError as e {
    switch e.Number {
        case 2:  ; File not found
            data := GetDefaultData()
        case 5:  ; Access denied
            throw  ; Re-throw - can't recover
        default:
            MsgBox "Unexpected OS error: " . e.Number
            data := ""
    }
}
```

### Validation Error Collection

```ahk
errors := []

try ValidateField1() catch ValueError as e {
    errors.Push({field: "Field1", error: e.Message})
}

try ValidateField2() catch ValueError as e {
    errors.Push({field: "Field2", error: e.Message})
}

if (errors.Length > 0) {
    msg := "Validation errors:`n"
    for err in errors
        msg .= err.field . ": " . err.error . "`n"
    MsgBox msg
}
```

### Retry on Transient Errors

```ahk
RetryOperation(operation, maxAttempts := 3) {
    attempts := 0
    loop maxAttempts {
        attempts++
        try {
            return operation()
        } catch OSError as e {
            if (e.Number = 32 && attempts < maxAttempts) {
                ; File in use - retry
                Sleep(1000)
                continue
            }
            throw  ; Give up or non-retryable error
        }
    }
}
```

---

## Common Mistakes

### 1. Catching Too Broad

```ahk
; WRONG - catches everything, even unexpected errors
try {
    ComplexOperation()
} catch {
    ; Silent failure - hides bugs
}

; RIGHT - catch specific types
try {
    ComplexOperation()
} catch TargetError {
    ; Expected - window might not exist
} catch as e {
    ; Unexpected - log and alert
    LogError(e)
    throw
}
```

### 2. Not Using `Extra` Property

```ahk
; WRONG - loses context
try {
    ProcessValue(input)
} catch ValueError as e {
    MsgBox "Invalid value"  ; Which value?
}

; RIGHT - include Extra
try {
    ProcessValue(input)
} catch ValueError as e {
    MsgBox "Invalid value: " . e.Extra
}
```

### 3. Forgetting Error Hierarchy

```ahk
; WRONG - order matters!
try {
    operation()
} catch ValueError as e {
    ; Handle ValueError
} catch IndexError as e {
    ; Never reached - IndexError extends ValueError!
}

; RIGHT - catch specific before generic
try {
    operation()
} catch IndexError as e {
    ; Handle IndexError specifically
} catch ValueError as e {
    ; Handle other ValueErrors
}
```

---

## Quick Reference Table

| Error Type | Common Cause | Check For | Recovery |
|------------|--------------|-----------|----------|
| **MemoryError** | Out of memory | N/A | Log and exit |
| **OSError** | File/API failure | `e.Number` | Retry or fallback |
| **TimeoutError** | SendMessage timeout | N/A | Retry shorter timeout |
| **TypeError** | Wrong type | `e.Extra` | Validate input |
| **ValueError** | Invalid value | `e.Extra` | Validate range |
| **IndexError** | Out of range | `e.Extra` | Check bounds |
| **ZeroDivisionError** | Divide by zero | N/A | Check denominator |
| **TargetError** | Target not found | N/A | Check existence first |
| **UnsetError** | Uninitialized var | N/A | Initialize or use `IsSet()` |
| **UnsetItemError** | Missing key | `e.Extra` | Use `.Has()` or `.Get()` |
| **PropertyError** | Bad property | `e.What` | Check property exists |
| **MethodError** | Missing method | `e.What` | Check method exists |

---

## See Also

- docs/manuals/ahk2/reference/type-system.md - Type-related errors
- docs/manuals/ahk2/reference/class-system.md - Custom error classes
- Official: [Error Object](https://www.autohotkey.com/docs/v2/lib/Error.htm)
- Official: [Try/Catch](https://www.autohotkey.com/docs/v2/lib/Try.htm)
