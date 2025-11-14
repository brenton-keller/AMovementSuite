# Explanation: Error Handling as Architecture

**Why error handling is a design decision, not just try/catch blocks**

Most developers treat error handling as tactical: wrap risky code in try/catch and move on. But error handling is fundamentally an **architectural decision** that affects debuggability, user experience, system resilience, and maintainability. This guide reveals the philosophy behind effective error handling in AutoHotkey v2: when to catch vs. propagate, how errors flow through call stacks, and why designing error boundaries is as important as designing your data structures.

## The Core Philosophy Shift

### From Tactical to Strategic

**Tactical thinking (common):**
- "This might fail, add try/catch"
- "Catch everything to prevent crashes"
- "Silent failures are safer than error dialogs"

**Strategic thinking (effective):**
- "Where should this error be handled in the architecture?"
- "What recovery strategies are possible at each layer?"
- "How do errors communicate problems to the right decision-maker?"

The difference: tactical handling scatters try/catch blocks reactively. Strategic handling designs error boundaries that make debugging easy and recovery natural.

## Error Flow and Propagation

### The Call Stack Journey

When an error occurs, it doesn't just stop execution—it travels **upward** through the call stack looking for a handler:

```
Function C: throw Error("Something failed")
   ↓ (no try/catch in C)
Function B: called C
   ↓ (no try/catch in B)
Function A: called B
   ↓ (has try/catch)
CAUGHT: Error handled in A

OR if no handlers exist:

Function C: throw Error("Something failed")
   ↓
Function B: (no handler)
   ↓
Function A: (no handler)
   ↓
Top Level: (no handler)
   ↓
OnError handlers: (in LIFO order)
   ↓
Default error dialog → Thread exits
```

**Key principle:** Errors bubble up until caught or reach OnError/error dialog.

### try/catch/finally Execution Order

**When NO error occurs:**
1. try block executes completely
2. catch block **SKIPPED**
3. else block executes (if present)
4. finally block executes (if present)

**When error IS thrown:**
1. try block stops at error point
2. Matching catch block executes
3. else block **SKIPPED**
4. finally block executes (if present)

**Critical:** finally **ALWAYS** runs—even with return, throw, or another error in catch.

**Exception:** Script directly terminated (ExitApp)

### Rethrowing: Transforming Errors Up the Stack

```ahk
try Example1()
catch Number as e
    MsgBox "Caught in main: " e

Example1() {
    try Example2()
    catch Number as e {
        if (e = 1)
            throw  ; Rethrow - propagates to caller
        MsgBox "Example1 caught " e
    }
}

Example2() {
    throw 1  ; Throws to Example1
}
```

Rethrowing allows you to:
- Log errors at intermediate layers
- Add context before propagating
- Filter which errors you handle vs. propagate

## When to Catch vs. Propagate: The Decision Tree

```
ERROR OCCURRED
│
├─ Can I meaningfully handle this error HERE?
│  ├─ YES → Catch and handle
│  │   └─ Can I fully recover?
│  │       ├─ YES → Catch, recover, log, continue
│  │       └─ NO → Catch, log, use fallback/default
│  │
│  └─ NO → Should I transform for caller?
│      ├─ YES → Catch, wrap in custom error, rethrow
│      └─ NO → Let it propagate
│
└─ Is this a programming bug vs runtime error?
   ├─ PROGRAMMING BUG → Let propagate (fail fast)
   │   Examples: null reference, logic error
   │
   └─ RUNTIME ERROR → Handle based on context
       ├─ USER INPUT → Catch, validate, friendly message
       ├─ TRANSIENT → Retry with backoff
       ├─ EXTERNAL SERVICE → Circuit breaker + retry
       ├─ CRITICAL RESOURCE → Catch, log, fail fast + alert
       └─ OPTIONAL FEATURE → Catch, log, graceful degradation
```

### Context-Specific Strategies

**User Input Validation - ALWAYS CATCH**

```ahk
GetUserAge() {
    loop {
        IB := InputBox("Enter your age:", "Age Verification")
        if (IB.Result = "Cancel")
            return ""

        try {
            age := Integer(IB.Value)
            if (age < 0 || age > 150)
                throw ValueError("Age must be between 0 and 150")
            return age
        } catch ValueError as e {
            MsgBox "Invalid input: " e.Message
            ; Loop continues - user can try again
        }
    }
}
```

**Why catch:** User input errors are expected and recoverable. The user needs immediate feedback to correct their mistake.

**File Operations - CATCH, LOG, FALLBACK**

```ahk
LoadConfig(path, useDefaults := true) {
    try {
        content := FileRead(path)
        return ParseConfig(content)
    } catch OSError as e {
        ErrorHandler.Log("Config load failed", path ": " e.Message)

        if (useDefaults) {
            ErrorHandler.Log("Using default configuration")
            return GetDefaultConfig()
        } else {
            throw  ; Rethrow if defaults not acceptable
        }
    }
}
```

**Why catch:** Config files might not exist or be corrupted. Falling back to defaults is often reasonable. But if defaults aren't acceptable (indicated by parameter), propagate to let caller decide.

**Network Operations - RETRY THEN CATCH**

```ahk
FetchData(url) {
    try {
        return RetryWithJitter(() => Download(url), 5, 2000, 30000)
    } catch as e {
        ErrorHandler.Log("All retry attempts failed for " url, e.Message)

        ; Try cache as fallback
        cached := GetCachedData(url)
        if (cached) {
            ErrorHandler.Log("Using cached data for " url)
            return cached
        }

        ; No cache - must fail
        throw Error("Cannot fetch data: all attempts failed and no cache available")
    }
}
```

**Why retry first:** Network failures are often transient. Exponential backoff handles temporary outages. Caching provides degraded service when the network is completely unavailable.

**Window Operations - CATCH TARGETERROR**

```ahk
SafeWinActivate(title, timeout := 5000) {
    try {
        if (!WinWait(title,, timeout/1000))
            throw TimeoutError("Window not found: " title)
        WinActivate(title)
        return true
    } catch TimeoutError as e {
        ErrorHandler.Log("Window activation failed", title)
        return false  ; Caller can handle gracefully
    } catch TargetError as e {
        ErrorHandler.Log("Window vanished during activation", title)
        return false
    }
}
```

**Why catch:** Window automation often deals with external processes. Windows may close unexpectedly. Returning false lets callers handle missing windows gracefully without crashing.

**Critical Operations - CATCH, LOG, FAIL FAST**

```ahk
InitializeDatabase(connectionString) {
    try {
        db := DatabaseConnect(connectionString)
        db.RunMigrations()
        return db
    } catch as e {
        ; Database init failure is critical - can't continue
        ErrorHandler.Log("CRITICAL: Database initialization failed", e.Message)
        MsgBox("Fatal error: Cannot initialize database.`n`n"
            "The application cannot start.`n`n"
            "Error: " e.Message, "Fatal Error", "IconX")
        ExitApp(1)  ; Fail fast
    }
}
```

**Why fail fast:** If the database is unavailable, the application can't function. Continuing would lead to cascading failures and data corruption. Better to fail immediately with a clear message.

### When to Propagate (Don't Catch)

**1. Cannot meaningfully handle**
```ahk
ProcessData(data) {
    ; Don't catch - let caller handle invalid data
    if (!ValidateData(data))
        throw ValueError("Invalid data format")

    ; Do processing
}
```

**2. Programming bug, not runtime error**
```ahk
Calculate(x, y, operation) {
    ; Don't catch - this indicates a bug in calling code
    if (!HasProp(operation, "Execute"))
        throw TypeError("operation must have Execute method")

    return operation.Execute(x, y)
}
```

**3. Caller better positioned to handle**
```ahk
; Low-level function - doesn't know business context
FetchUserData(userId) {
    ; Let OSError propagate - caller knows if this is critical
    return FileRead("users\" userId ".json")
}

; High-level function - has context
ShowUserProfile(userId) {
    try {
        data := FetchUserData(userId)
        DisplayProfile(data)
    } catch OSError {
        MsgBox "User profile not found. Using default profile."
        DisplayProfile(GetDefaultProfile())
    }
}
```

## OnError: The Global Safety Net

### How OnError Interacts with try/catch

**Critical distinction:** try/catch has priority over OnError.

```ahk
OnError(GlobalHandler)

try {
    x := 1/0  ; Caught by try/catch
} catch as e {
    MsgBox "Caught locally: " e.Message
    ; OnError NOT called - error was caught
}

y := 1/0  ; NOT in try/catch
          ; → OnError IS called
          ; → GlobalHandler executes

GlobalHandler(exception, mode) {
    FileAppend "Unhandled error: " exception.Message "`n", "error.log"
    return 1  ; Suppress dialog
}
```

**OnError is called ONLY for unhandled errors** - errors that escape all try/catch blocks.

### Return Values and Their Effects

**Return 1:** "I handled this"
- Error suppressed (no dialog)
- Thread exits
- No further OnError handlers called

**Return 0:** "Pass to next handler"
- Continue to next OnError handler
- If no more handlers, show default error dialog
- Thread exits

**Return -1:** "Try to continue" (Mode = "Return" only)
- Thread continues execution
- Extremely rare - only for continuable errors
- If Mode != "Return", behaves like return 0

### Multiple Handler Registration (LIFO Order)

```ahk
OnError(Handler1)  ; Registered first
OnError(Handler2)  ; Registered second
OnError(Handler3)  ; Registered third

; Execution order when error occurs:
; Handler3 → Handler2 → Handler1 (Last In, First Out)
```

### Production-Ready OnError Example

```ahk
OnError(ProductionErrorHandler)

ProductionErrorHandler(thrown, mode) {
    ; 1. Log to file
    timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
    logEntry := Format(
        "[{1}] {2} in {3}() at line {4}: {5}`nStack: {6}`n`n",
        timestamp, Type(thrown), thrown.What,
        thrown.Line, thrown.Message, thrown.Stack
    )
    FileAppend logEntry, A_ScriptDir "\error.log"

    ; 2. Determine severity
    isCritical := (thrown is MemoryError) || (thrown is OSError)

    ; 3. User notification
    if (isCritical) {
        MsgBox Format("A critical error occurred: {1}`n`n"
            "Details have been logged. Please contact support.",
            thrown.Message), "Error", "IconX T10"
    }

    ; 4. Decide recovery
    if (mode = "Return" && !isCritical)
        return -1  ; Try to continue for non-critical errors

    return 1  ; Suppress dialog, exit thread
}
```

## Timer Callback Error Handling: The Thread Problem

### Why External try/catch Doesn't Work

```ahk
; THIS WILL NEVER CATCH THE ERROR
try {
    SetTimer () => ThrowError(), -100
    Sleep 200
} catch {
    MsgBox "Never caught!"  ; Never executes
}

ThrowError() {
    throw Error("Timer error!")
}
```

### The Fundamental Problem: Thread Execution Context

AutoHotkey uses a pseudo-threading model. Each timer, hotkey, and GUI event creates a new "thread" (execution context).

```
Thread 1 (main):
  [0ms]   try { ... }
  [0ms]     SetTimer(callback, -100)  ✓ Success
  [0ms]     Sleep(200)                ✓ Success
  [200ms] }  ✓ Try block completes with NO error
  [200ms] catch { ... }  ✗ Never executes

Thread 2 (timer - fires at 100ms):
  [100ms] callback executes
  [100ms] throw Error()  ✗ Error occurs HERE in Thread 2
  [100ms] No try/catch in this thread!
  [100ms] → OnError handlers called
  [100ms] → Error dialog shown
  [100ms] → Thread 2 exits

Thread 1 continues unaware.
```

**The rule:** Errors can ONLY be caught by try/catch in the SAME thread where they occur.

### Solution 1: try/catch INSIDE Timer Callback

```ahk
SetTimer () => SafeTimerCallback(), -100

SafeTimerCallback() {
    try {
        ThrowError()  ; Error caught here
    } catch as e {
        MsgBox "Caught in timer: " e.Message
    }
}
```

### Solution 2: Use OnError for Global Timer Error Handling

```ahk
OnError(HandleTimerError)

SetTimer () => ThrowError(), -100

HandleTimerError(thrown, mode) {
    ; OnError catches errors from ALL threads
    FileAppend Format("Timer error: {1} at {2}`n",
        thrown.Message, FormatTime(, "HH:mm:ss")), "timer_errors.log"
    return 1  ; Suppress dialog
}
```

### Solution 3: Design for Error Resilience (Recommended)

```ahk
; Best practice: wrap ALL timer logic in try/catch
SetTimer () => {
    try {
        ; All timer work here
        DoSomeWork()
        CheckStatus()
        UpdateUI()
    } catch as e {
        ; Log error, continue timer operation
        FileAppend FormatTime() " - Timer error: " e.Message "`n", "errors.log"
        ; Timer continues to run despite error
    }
}, 1000  ; Repeating timer
```

## Error Handling Performance: Debunking Myths

### MYTH: "try/catch is expensive"

**REALITY:** Direct quote from Lexikos (AHK v2 lead developer):

> "Try/catch is implemented very similarly to If, Loop, and other control flow constructs... **I would expect try to execute faster than if true**. A more efficient implementation would not impose any runtime cost until an exception is thrown... **there is negligible 'setup' cost**."

**Performance data:**

**When NO error occurs (99% of execution):**
- try/catch overhead: **Negligible**
- try may actually be **slightly faster** than if statements
- Zero-cost abstraction in the happy path

**Why try is faster than if:**
- if statement: Must evaluate boolean condition every time
- try statement: No parameters to evaluate, just checks return status

**When errors ARE thrown (rare cases):**
- Creating Error object: Expensive
- Generating stack trace: Expensive (depends on call stack depth)
- 10-650x slower than normal execution
- **But this should be rare** - that's why they're "exceptions"

### What's Actually Expensive

**EXPENSIVE:**
- Creating Error objects with stack traces
- Throwing exceptions frequently in loops
- Deep call stacks when error occurs

**NOT EXPENSIVE:**
- Wrapping code in try blocks
- Having catch blocks that don't execute
- Using try/catch instead of if for error handling

### Performance Best Practices

**DO use try/catch:**
- For truly exceptional conditions (rare errors)
- Around file I/O, network operations, Windows API calls
- For code clarity and maintainability
- In most normal code paths

**AVOID throwing exceptions:**
- In tight performance-critical loops (e.g., FFT inner loop)
- For expected/common control flow
- For input validation where failures are frequent

**Example:**
```ahk
; GOOD: Error is rare - use try/catch
try {
    FileRead("config.ini")
} catch OSError as err {
    LoadDefaultConfig()
}

; BAD: Error is common - check first
for key in manyKeys {
    ; DON'T: throw UnsetItemError 10,000 times
    try {
        value := myMap[key]
    } catch UnsetItemError {
        value := default
    }

    ; DO: Check first to avoid throwing
    value := myMap.Has(key) ? myMap[key] : default
}
```

**Quote from Lexikos:**
> "If the key is commonly not found, checking for it first is probably more efficient - **not because of any penalty imposed by try-catch, but because of the cost of constructing a KeyError, complete with stack trace**."

## Retry Logic: Transient Failure Patterns

Transient failures are common (network blips, file locks, resource contention). Retry logic handles these gracefully.

### Pattern: Exponential Backoff with Jitter (Recommended)

```ahk
RetryWithJitter(operation, maxRetries := 5, baseDelay := 1000, maxDelay := 32000) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            return operation()
        } catch as e {
            if (attempts >= maxRetries)
                throw e

            ; Exponential backoff
            exponentialDelay := baseDelay * (2 ** (attempts - 1))
            cappedDelay := Min(exponentialDelay, maxDelay)

            ; Add random jitter (±25%)
            jitter := Random(-0.25 * cappedDelay, 0.25 * cappedDelay)
            actualDelay := Integer(cappedDelay + jitter)

            Sleep(actualDelay)
        }
    }
}

; Usage
data := RetryWithJitter(() => FetchFromAPI("https://api.example.com/data"), 5, 2000, 30000)
```

**Why jitter matters:** Prevents "thundering herd" when multiple clients retry simultaneously. Randomization spreads out retry attempts.

### Pattern: Conditional Retry (Specific Error Types)

```ahk
RetryOnSpecificErrors(operation, maxRetries := 3, retryableErrors*) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            return operation()
        } catch as e {
            ; Check if error is retryable
            isRetryable := false
            for errorType in retryableErrors {
                if (e is %errorType%) {
                    isRetryable := true
                    break
                }
            }

            if (!isRetryable || attempts >= maxRetries)
                throw e  ; Don't retry this error type

            delay := 1000 * (2 ** (attempts - 1))
            Sleep(delay)
        }
    }
}

; Usage - retry only transient errors
result := RetryOnSpecificErrors(
    () => NetworkOperation(),
    5,
    OSError,      ; Retry OS/network errors
    TimeoutError  ; Retry timeouts
)
; But don't retry ValueError (bad input data)
```

### Pattern: Circuit Breaker (Fail Fast After Multiple Failures)

When a service is consistently failing, stop hammering it. The circuit breaker pattern prevents cascading failures and allows recovery time.

```ahk
class CircuitBreaker {
    static State := {CLOSED: 0, OPEN: 1, HALF_OPEN: 2}

    __New(failureThreshold := 5, timeout := 60000, successThreshold := 2) {
        this.failureThreshold := failureThreshold
        this.timeout := timeout
        this.successThreshold := successThreshold

        this.failureCount := 0
        this.successCount := 0
        this.state := CircuitBreaker.State.CLOSED
        this.lastFailureTime := 0
    }

    Call(operation) {
        ; If circuit open, check if timeout elapsed
        if (this.state == CircuitBreaker.State.OPEN) {
            if (A_TickCount - this.lastFailureTime > this.timeout) {
                this.state := CircuitBreaker.State.HALF_OPEN
                this.successCount := 0
            } else {
                throw Error("Circuit breaker OPEN - service unavailable")
            }
        }

        ; Try operation
        try {
            result := operation()
            this.OnSuccess()
            return result
        } catch as e {
            this.OnFailure()
            throw e
        }
    }

    OnSuccess() {
        this.failureCount := 0

        if (this.state == CircuitBreaker.State.HALF_OPEN) {
            this.successCount++
            if (this.successCount >= this.successThreshold)
                this.state := CircuitBreaker.State.CLOSED
        }
    }

    OnFailure() {
        this.lastFailureTime := A_TickCount
        this.failureCount++

        if (this.failureCount >= this.failureThreshold)
            this.state := CircuitBreaker.State.OPEN
    }
}
```

## Centralized Error Handler: Production Implementation

```ahk
class ErrorHandler {
    static instance := ""
    static logFile := ""
    static errorCount := Map()
    static debugMode := false
    static maxErrorsPerMinute := 10
    static lastNotificationTime := 0
    static notificationCooldown := 60000

    ; Initialize singleton
    static Init(logPath := "", debug := false) {
        if (this.instance != "")
            return this.instance

        this.logFile := logPath ? logPath : A_ScriptDir "\error_log.txt"
        this.debugMode := debug
        this.instance := this

        ; Register global error handler
        OnError(ObjBindMethod(this, "HandleError"))

        return this
    }

    ; Main error handling
    static HandleError(exception, mode) {
        timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
        errorKey := exception.File ":" exception.Line

        ; Track error frequency for rate limiting
        if (!this.errorCount.Has(errorKey))
            this.errorCount[errorKey] := {count: 0, firstSeen: A_TickCount}

        errorInfo := this.errorCount[errorKey]
        errorInfo.count++

        ; Rate limiting
        timeSinceFirst := A_TickCount - errorInfo.firstSeen
        if (timeSinceFirst < 60000 && errorInfo.count > this.maxErrorsPerMinute) {
            this.LogToFile(timestamp, "RATE_LIMITED", exception)
            return 1
        }

        ; Always log
        this.LogToFile(timestamp, "ERROR", exception)

        ; Determine if user should be notified
        shouldNotify := this.ShouldNotifyUser(exception)

        if (shouldNotify && this.debugMode) {
            this.ShowDebugDialog(exception)
            return 0
        }

        if (shouldNotify && !this.debugMode) {
            this.ShowUserFriendlyError(exception)
            return 1
        }

        return 1
    }

    static LogToFile(timestamp, level, exception) {
        logEntry := Format(
            "[{1}] {2} - {3}: {4}`n  Type: {5}`n  File: {6}`n  Line: {7}`n",
            timestamp, level, Type(exception), exception.Message,
            Type(exception), exception.File, exception.Line
        )

        try {
            FileAppend(logEntry, this.logFile)
        }
    }

    static ShouldNotifyUser(exception) {
        ; Respect cooldown
        if (A_TickCount - this.lastNotificationTime < this.notificationCooldown)
            return false

        ; Always notify for critical errors
        if (exception is MemoryError || exception is OSError) {
            this.lastNotificationTime := A_TickCount
            return true
        }

        return false
    }

    static ShowUserFriendlyError(exception) {
        this.lastNotificationTime := A_TickCount

        ; Customize message based on error type
        if (exception is OSError)
            userMsg := "A system error occurred. Please check file permissions or system resources."
        else if (exception is TargetError)
            userMsg := "Could not find the target window or control. It may have closed."
        else if (exception is ValueError || exception is TypeError)
            userMsg := "Invalid data was encountered. Please check your input."
        else
            userMsg := "An unexpected error occurred."

        userMsg .= "`n`nDetails have been logged. Contact support if this persists."

        MsgBox(userMsg, "Application Error", "IconX T10")
    }
}
```

## Key Insights

**Insight 1: Errors are architectural decisions**
Where you catch errors determines recovery strategies, debugging experience, and system resilience.

**Insight 2: try/catch has negligible overhead**
The myth that try/catch is expensive is false. Wrapping code in try is essentially free when no error occurs.

**Insight 3: Error flow follows call stack**
Errors bubble up until caught or reach OnError. Understanding this flow determines where to place handlers.

**Insight 4: Context determines handling strategy**
User input errors need immediate feedback. Network errors need retry. Critical errors need fail-fast.

**Insight 5: Timer callbacks create new threads**
Errors in timer callbacks can't be caught by external try/catch. Use internal handling or OnError.

**Insight 6: Retry with backoff handles transient failures**
Network blips and resource contention are temporary. Exponential backoff with jitter prevents thundering herd.

**Insight 7: Circuit breakers prevent cascading failures**
When a service is down, failing fast is better than continuous retries that worsen the problem.

## Cross-References

Related topics:
- docs/manuals/ahk2/reference/error-objects.md - Error object properties
- docs/manuals/ahk2/how-to/retry-patterns.md - Retry implementation patterns
- docs/manuals/ahk2/reference/onerror.md - OnError reference
- docs/manuals/ahk2/explanation/type-coercion-logic.md - Type-related errors
