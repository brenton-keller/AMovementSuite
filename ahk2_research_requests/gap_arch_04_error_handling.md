# AHK2 Research Request: Error Handling Architecture

## What I Need to Learn

My codebase has 18 scattered try/catch blocks - some log errors, some fail silently, some show MsgBox. I have `ErrorHandler.ahk2` that's unused. I need a **centralized error handling strategy** that works consistently across the whole application.

## My Current Understanding (Challenge This!)

I believe:
- try/catch is expensive (avoid in loops)
- Silent failure (`catch {}`) is sometimes OK
- Error objects just have a Message property
- You can't catch different error types separately
- Errors in timer callbacks can be caught from outside

**Which of these are wrong?**

## The Core Problem

My actual code patterns:
```ahk
// Pattern 1: Silent failure
try {
    WinGetPos &x, &y, &w, &h, "ahk_id " hwnd
} catch {
    continue  // Error lost forever
}

// Pattern 2: User-facing MsgBox
try {
    result := Calculate(expression)
} catch as err {
    MsgBox "Error: " err.Message  // Annoying for users
}

// Pattern 3: Detailed logging
try {
    FileRead(path)
} catch Error as e {
    log := "Error in " A_LineFile ":" A_LineNumber "`n" e.Message
    FileAppend log, "errors.log"
}

// I have ErrorHandler.ahk2 but don't know how to use it consistently
```

## Specific Questions

1. **Error Types**: What's the error hierarchy? Can I catch OSError separately from TypeError?

2. **Error Properties**: What's in an Error object besides Message? Stack traces? File/line info?

3. **OnError Handler**: How does OnError work? Can I have multiple handlers? What's the execution order?

4. **try/catch/finally**: What's the execution flow? Does finally run if I return from try? If exception isn't caught?

5. **Timer Errors**: Why can't I catch errors from timer callbacks? How DO I handle them?

6. **Performance**: What's the REAL overhead of try/catch when no error occurs?

## What I Need to Build

A centralized error handler that:
```ahk
class ErrorHandler {
    // Logs all errors automatically
    // Shows user-friendly message in production
    // Shows detailed stack trace in debug mode
    // Implements retry logic for transient failures
    // Tracks error frequency (don't spam user)
    // Gracefully degrades on errors
}

// Usage everywhere:
result := ErrorHandler.Try(() => RiskyOperation(), fallbackValue)
```

## How to Write This Guide

### 1. Error Flow Visualization

Show the complete error flow:
```
Code throws error
  ↓
Matching catch block?
  ├─ Yes → Execute catch, then finally
  └─ No → Execute finally, then bubble up
  ↓
Parent function has try/catch?
  ├─ Yes → Repeat above
  └─ No → Check OnError handlers
  ↓
OnError returns 1?
  ├─ Yes → Exit thread
  └─ No → Show default error dialog
```

### 2. Error Type Hierarchy with Examples

Show the COMPLETE hierarchy:
```
Error (base)
├── MemoryError     → [when does this occur?]
├── OSError         → [example: file not found]
│   └── .Number property = Windows error code
├── TypeError       → [example: "hello" + 5]
├── ValueError      → [example: invalid parameter]
├── TargetError     → [example: window doesn't exist]
├── UnsetError      → [example: accessing uninitialized var]
└── MemberError
    ├── PropertyError
    └── MethodError
```

For each, show:
- When it's thrown
- What the properties contain
- How to catch it specifically
- How to handle it appropriately

### 3. Progressive Error Handling

**Level 1: Basic try/catch**
```ahk
try {
    result := RiskyOperation()
} catch Error as e {
    MsgBox e.Message
}
```

**Level 2: Specific error types**
```ahk
try {
    FileRead(path)
} catch OSError as e {
    if (e.Number = 2)  // File not found
        MsgBox "File doesn't exist"
    else
        throw  // Re-throw other OS errors
}
```

**Level 3: Finally for cleanup**
```ahk
file := ""
try {
    file := FileOpen(path, "r")
    ProcessFile(file)
} catch Error as e {
    LogError(e)
} finally {
    if (file)
        file.Close()  // Always cleanup
}
```

**Level 4: Custom error types**
```ahk
class WindowError extends Error {
    __New(message, hwnd := 0) {
        super.__New(message)
        this.WindowID := hwnd
    }
}

throw WindowError("Window invalid", mwin)
```

**Level 5: Centralized handler**
```ahk
class ErrorHandler {
    static Try(operation, fallback := "", context := "") {
        try {
            return operation()
        } catch Error as e {
            this.Log(e, context)
            return fallback
        }
    }
}
```

### 4. OnError Deep Dive

Explain OnError completely:
```ahk
OnError(HandlerFunction)

HandlerFunction(thrown, mode) {
    /*
    thrown = Error object
    mode = "Exit" or "ExitApp"

    Return values and their meaning:
    0 or omitted → Show dialog, call next handler
    1 → Suppress dialog, exit thread
    -1 → Suppress dialog, CONTINUE (dangerous!)
    */
}
```

Show examples of each return value and when to use them.

### 5. The Timer Callback Problem

Explain WHY this doesn't work:
```ahk
try {
    SetTimer () => ThrowError(), -100
    Sleep 200
} catch {
    MsgBox "Never caught!"  // Why?
}
```

Then show the RIGHT way:
```ahk
SafeTimer() {
    try {
        RiskyOperation()
    } catch Error as e {
        ErrorHandler.Log(e)  // Handle inside timer
    }
}

SetTimer SafeTimer, -100
```

### 6. Retry Logic Patterns

Show increasing sophistication:

**Naive retry:**
```ahk
Loop 3 {
    try {
        return Download(url)
    } catch {
        Sleep 1000
    }
}
throw Error("Failed after 3 attempts")
```

**Exponential backoff:**
```ahk
RetryWithBackoff(operation, maxAttempts := 3) {
    Loop maxAttempts {
        try {
            return operation()
        } catch Error as e {
            if (A_Index = maxAttempts)
                throw  // Last attempt failed

            delay := (2 ** (A_Index - 1)) * 1000  // 1s, 2s, 4s
            Sleep delay
        }
    }
}
```

**Production-ready with logging:**
```ahk
class RetryPolicy {
    // Complete implementation with:
    // - Exponential backoff with jitter
    // - Retry only on specific error types
    // - Callback for each retry
    // - Circuit breaker pattern
}
```

### 7. Performance Testing

Test the claims:
```ahk
; Test 1: try/catch overhead when NO error
Loop 100000 {
    try {
        x := 1 + 1
    }
}
// Time: ???ms

; Test 2: same code without try/catch
Loop 100000 {
    x := 1 + 1
}
// Time: ???ms

; Overhead: ???%
```

Then test WITH errors thrown to show that's where the real cost is.

### 8. The Centralized Handler Pattern

Show COMPLETE, production-ready implementation:
```ahk
class ErrorHandler {
    static logFile := A_ScriptDir "\errors.log"
    static DEBUG_MODE := !A_IsCompiled

    static Initialize() {
        ; Set up global handler
        OnError(ErrorHandler.Handle.Bind(ErrorHandler))
    }

    static Handle(thrown, mode) {
        ; Implementation that:
        ; - Logs all errors
        ; - Shows different UI for debug vs production
        ; - Tracks error frequency
        ; - Implements rate limiting on notifications
    }

    static Try(operation, fallback := "", context := "") {
        ; Wrapper for safe operation execution
    }

    static Log(err, context := "") {
        ; Detailed logging implementation
    }
}

; How to use throughout codebase
```

### 9. When to Catch vs Propagate

Decision tree:
```
Error occurred
  ↓
Can you fix it?
├─ Yes → Catch, fix, continue
└─ No ↓
  ↓
Can you provide fallback?
├─ Yes → Catch, use fallback
└─ No ↓
  ↓
Should you retry?
├─ Yes → Catch, retry with backoff
└─ No ↓
  ↓
Is it expected?
├─ Yes → Catch, log, maybe notify user
└─ No ↓
  ↓
LET IT PROPAGATE → Caller decides
```

### 10. Common Mistakes with Consequences

**Mistake 1: Swallowing errors**
```ahk
try {
    CriticalOperation()
} catch {
    // Silent failure - lost data, no idea why
}

// Consequence: User loses work, no error message, can't debug
```

**Mistake 2: Too broad catch**
```ahk
try {
    FileRead(path)
    data := ProcessData(content)  // Bug in this function
} catch {
    MsgBox "File not found"  // Wrong! Could be processing error
}
```

**Mistake 3: Not cleaning up**
```ahk
file := FileOpen(path)
try {
    ProcessFile(file)
} catch {
    return  // File handle leaked!
}
```

## Success Criteria

After this guide:
1. ✓ Implement centralized ErrorHandler class used throughout codebase
2. ✓ Understand complete error hierarchy and catch specific types
3. ✓ Know when to catch vs propagate
4. ✓ Implement retry logic with exponential backoff
5. ✓ Handle errors in timer callbacks correctly
6. ✓ Log errors without annoying users

## Most Important

I don't want to just learn try/catch syntax. I want to understand **error handling as an architectural decision** - designing systems that fail gracefully, preserve user data, and provide actionable error information.

Show me how experienced developers think about errors, not just how to catch them.
