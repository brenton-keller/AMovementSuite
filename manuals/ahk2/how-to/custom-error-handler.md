# How-To: Set Up Centralized Error Handling

## Problem Statement

Scattered try-catch blocks throughout your codebase lead to inconsistent error handling, lost debugging information, and poor user experience. Errors might be silently swallowed, logged inconsistently, or spam users with message boxes. You need a centralized system that catches all errors, logs them consistently, provides appropriate user feedback, and enables debugging.

## Basic Global Error Handler

AutoHotkey v2's `OnError` function provides a global safety net for unhandled errors:

```ahk
; Register global error handler at script start
OnError(GlobalErrorHandler)

GlobalErrorHandler(exception, mode) {
    ; Log the error
    timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
    logEntry := Format("[{1}] {2}: {3}`n", timestamp, Type(exception), exception.Message)
    FileAppend logEntry, "error.log"

    ; Show user-friendly message
    MsgBox Format("An error occurred: {1}", exception.Message), "Error", "IconX"

    return 1  ; Suppress default error dialog
}

; Your code - unhandled errors will be caught
result := 10 / 0  ; Triggers OnError handler
```

**Key parameters:**
- `exception`: Error object with properties (Message, What, File, Line, Stack, etc.)
- `mode`: String indicating error type ("Exit" or "Return")

**Return values:**
- `1`: "I handled this" - suppresses error dialog, thread exits
- `0`: "Pass to next handler" - continues to next OnError handler or default dialog
- `-1`: "Try to continue" (mode = "Return" only) - thread continues execution

## Production-Ready Error Handler Class

```ahk
#Requires AutoHotkey v2.0

class ErrorHandler {
    static instance := ""
    static logFile := ""
    static errorCount := Map()
    static debugMode := false
    static maxErrorsPerMinute := 10
    static lastNotificationTime := 0
    static notificationCooldown := 60000  ; 1 minute

    ; Initialize singleton
    static Init(logPath := "", debug := false) {
        if (this.instance != "")
            return this.instance

        this.logFile := logPath ? logPath : A_ScriptDir "\error_log.txt"
        this.debugMode := debug
        this.instance := this

        ; Register global error handler
        OnError(ObjBindMethod(this, "HandleError"))

        ; Periodic cleanup of error counts
        SetTimer(ObjBindMethod(this, "CleanupErrorCounts"), 300000)  ; Every 5 min

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

        ; Rate limiting - stop logging after threshold
        timeSinceFirst := A_TickCount - errorInfo.firstSeen
        if (timeSinceFirst < 60000 && errorInfo.count > this.maxErrorsPerMinute) {
            this.LogToFile(timestamp, "RATE_LIMITED", exception)
            return 1  ; Suppress further notifications
        }

        ; Always log the error
        this.LogToFile(timestamp, "ERROR", exception)

        ; Determine if user should be notified
        shouldNotify := this.ShouldNotifyUser(exception)

        ; Debug mode: show detailed error dialog
        if (shouldNotify && this.debugMode) {
            this.ShowDebugDialog(exception)
            return 0  ; Show default error dialog too
        }

        ; Production mode: user-friendly notification
        if (shouldNotify && !this.debugMode) {
            this.ShowUserFriendlyError(exception)
            return 1  ; Suppress default dialog
        }

        return 1  ; Suppress dialog by default
    }

    ; Write to log file
    static LogToFile(timestamp, level, exception) {
        logEntry := Format(
            "[{1}] {2} - {3}: {4}`n"
            "  Type: {5}`n"
            "  File: {6}`n"
            "  Line: {7}`n"
            "  What: {8}`n"
            "  Stack:`n{9}`n"
            "----------------------------------------`n",
            timestamp, level, Type(exception), exception.Message,
            Type(exception), exception.File, exception.Line,
            exception.What, this.IndentStack(exception.Stack)
        )

        try {
            FileAppend(logEntry, this.logFile)
        } catch {
            ; Failsafe: can't write to log
            OutputDebug("ErrorHandler: Failed to write to log file")
        }
    }

    ; Indent stack trace for readability
    static IndentStack(stack) {
        lines := StrSplit(stack, "`n")
        result := ""
        for line in lines
            result .= "    " line "`n"
        return RTrim(result, "`n")
    }

    ; Decide if user notification warranted
    static ShouldNotifyUser(exception) {
        ; Don't spam - respect cooldown
        if (A_TickCount - this.lastNotificationTime < this.notificationCooldown)
            return false

        ; Always notify for critical errors
        if (exception is MemoryError || exception is OSError) {
            this.lastNotificationTime := A_TickCount
            return true
        }

        ; Notify for errors in main execution (not timers/callbacks)
        if (InStr(exception.Stack, "Auto-execute")) {
            this.lastNotificationTime := A_TickCount
            return true
        }

        return false
    }

    ; Debug dialog with full error details
    static ShowDebugDialog(exception) {
        msg := Format(
            "═══ DEBUG ERROR REPORT ═══`n`n"
            "Error: {1}`n"
            "Type: {2}`n"
            "File: {3}`n"
            "Line: {4}`n"
            "Function: {5}`n"
            "Extra: {6}`n`n"
            "Call Stack:`n{7}",
            exception.Message, Type(exception),
            exception.File, exception.Line,
            exception.What, exception.Extra,
            exception.Stack
        )
        MsgBox(msg, "Debug Error", "Icon! T0")
    }

    ; User-friendly error notification
    static ShowUserFriendlyError(exception) {
        if (A_TickCount - this.lastNotificationTime < this.notificationCooldown)
            return

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

    ; Manual logging from caught exceptions
    static Log(message, context := "") {
        timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
        logEntry := Format("[{1}] INFO: {2}", timestamp, message)
        if (context)
            logEntry .= "`n  Context: " context
        logEntry .= "`n"

        try {
            FileAppend(logEntry, this.logFile)
        }
    }

    ; Clean up old error frequency tracking
    static CleanupErrorCounts() {
        currentTime := A_TickCount
        for key, info in this.errorCount {
            if (currentTime - info.firstSeen > 300000)  ; 5 minutes old
                this.errorCount.Delete(key)
        }
    }

    ; Manual error reporting with retry
    static LogWithRetry(message, exception, retryContext := "") {
        this.Log(Format("RETRY: {1} - {2}", message, exception.Message), retryContext)
    }
}
```

## Initialization and Configuration

At the start of your script:

```ahk
#Requires AutoHotkey v2.0

; Determine if in debug mode
IS_DEBUG := A_IsCompiled ? false : true  ; Debug for .ahk, production for .exe

; Initialize error handling
ErrorHandler.Init(A_ScriptDir "\logs\error.log", IS_DEBUG)

; Your script code here
```

**Configuration options:**
- `logPath`: Where to write error logs (default: script directory)
- `debug`: Enable detailed error dialogs during development

## Usage Patterns

### Pattern 1: Let Errors Propagate to Global Handler

For unexpected errors that indicate bugs:

```ahk
; Don't catch programming errors - let them reach OnError
result := CalculateValue(input)  ; TypeError if input is wrong type
ProcessResult(result)
```

### Pattern 2: Catch Expected Errors Locally

For errors you can handle meaningfully:

```ahk
try {
    WinActivate("ahk_exe notepad.exe")
} catch TargetError as e {
    ErrorHandler.Log("Window not found", "notepad.exe")
    MsgBox "Please open Notepad first"
    return
}
```

### Pattern 3: Catch, Log, and Rethrow

For errors you want to track but can't handle:

```ahk
try {
    data := LoadCriticalConfig()
} catch as e {
    ErrorHandler.Log("Critical config load failed", e.Message)
    throw e  ; Let global handler deal with it
}
```

### Pattern 4: Catch, Transform, and Throw Custom Error

For adding context to errors:

```ahk
class ConfigError extends Error { }

LoadConfig(section) {
    try {
        return IniRead("app.ini", section, "Value")
    } catch as e {
        ErrorHandler.Log("Config read failed", section)
        throw ConfigError("Failed to load section: " section)
    }
}
```

## Integration with Retry Logic

Combine centralized logging with retry patterns:

```ahk
RetryWithLogging(operation, operationName, maxRetries := 3) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            result := operation()
            if (attempts > 1)
                ErrorHandler.Log(Format("{1} succeeded on attempt {2}", operationName, attempts))
            return result

        } catch as e {
            ErrorHandler.LogWithRetry(
                Format("{1} failed on attempt {2}/{3}",
                    operationName, attempts, maxRetries),
                e,
                Format("Attempt {1}, Error: {2}", attempts, Type(e))
            )

            if (attempts >= maxRetries)
                throw e  ; Let global handler deal with final failure

            delay := 1000 * (2 ** (attempts - 1))
            Sleep(delay)
        }
    }
}

; Usage
config := RetryWithLogging(() => FileRead("config.ini"), "Load config", 3)
```

## Complete Example: Application with Centralized Error Handling

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

; Initialize error handling first
IS_DEBUG := !A_IsCompiled
ErrorHandler.Init(A_ScriptDir "\logs\app_errors.log", IS_DEBUG)

; Create GUI
MyGui := Gui()
MyGui.Add("Text", , "Enter a number to divide 100:")
editBox := MyGui.Add("Edit", "w200 vUserInput")
MyGui.Add("Button", "Default", "Calculate").OnEvent("Click", Calculate)
MyGui.Show()

Calculate(*) {
    userInput := editBox.Value

    ; Validate input locally
    if (userInput = "") {
        MsgBox "Please enter a number"
        return
    }

    if !IsNumber(userInput) {
        ErrorHandler.Log("User input validation failed", "Input: " userInput)
        MsgBox "Please enter a valid number"
        return
    }

    try {
        divisor := Float(userInput)

        ; This will throw if divisor is 0
        result := 100 / divisor

        MsgBox "Result: " result

    } catch ZeroDivisionError as e {
        ; Handle expected error locally
        ErrorHandler.Log("Division by zero attempted", "User input: " userInput)
        MsgBox "Cannot divide by zero!"

    } catch as e {
        ; Unexpected error - let global handler deal with it
        ErrorHandler.Log("Unexpected calculation error", Type(e) ": " e.Message)
        throw e
    }
}

; Load settings with error handling
LoadSettings() {
    try {
        return RetryWithLogging(() => FileRead("settings.ini"), "Load settings", 3)
    } catch as e {
        ErrorHandler.Log("CRITICAL: Cannot load settings", e.Message)
        MsgBox "Failed to load settings. Using defaults.", "Warning", "Iconi"
        return ""  ; Use defaults
    }
}

settings := LoadSettings()
```

## Troubleshooting

### Errors not being logged

**Problem:** OnError handler not being called.

**Cause:** Errors are being caught by try-catch before reaching OnError.

**Solution:** Only catch errors you can handle. Let unexpected errors propagate:

```ahk
; ❌ Wrong - catches everything
try {
    DoSomething()
} catch {
    ; Error disappears
}

; ✅ Right - only catch specific expected errors
try {
    DoSomething()
} catch TargetError {
    ; Handle this specific error
}
; All other errors propagate to OnError
```

### Error spam in logs

**Problem:** Same error repeating thousands of times.

**Solution:** The ErrorHandler class includes rate limiting. Adjust threshold:

```ahk
ErrorHandler.maxErrorsPerMinute := 5  ; Lower threshold
```

### Can't see error details during development

**Problem:** Production error messages hide details.

**Solution:** Enable debug mode:

```ahk
ErrorHandler.Init(A_ScriptDir "\error.log", true)  ; Debug mode ON
```

### Log file growing too large

**Problem:** Error log accumulating over months.

**Solution:** Implement log rotation:

```ahk
RotateLogFile(logPath, maxSize := 1048576) {  ; 1MB default
    if !FileExist(logPath)
        return

    ; Check file size
    size := FileGetSize(logPath)
    if (size < maxSize)
        return

    ; Rotate: app.log → app.log.1, app.log.1 → app.log.2, etc.
    if FileExist(logPath ".3")
        FileDelete(logPath ".3")
    if FileExist(logPath ".2")
        FileMove(logPath ".2", logPath ".3")
    if FileExist(logPath ".1")
        FileMove(logPath ".1", logPath ".2")
    FileMove(logPath, logPath ".1")
}

; Call at script startup
RotateLogFile(A_ScriptDir "\error.log")
```

### Timer/callback errors not caught

**Problem:** OnError not catching errors in timers.

**Cause:** You're using try-catch outside the timer, which doesn't work due to separate thread contexts.

**Solution:** OnError DOES catch timer errors. Just don't wrap SetTimer in try-catch:

```ahk
; ❌ Wrong - try-catch won't catch timer errors
try {
    SetTimer(() => ThrowError(), -100)
} catch {
    MsgBox "Never caught!"
}

; ✅ Right - OnError catches timer errors
OnError(ErrorHandler.HandleError)  ; Global handler
SetTimer(() => ThrowError(), -100)  ; Error will be caught by OnError
```

## Performance Considerations

**Error handling overhead:**
- OnError registration: One-time, negligible
- Normal execution (no errors): Zero overhead
- When error occurs: ~1-5ms for logging

**Log file I/O:**
- FileAppend is buffered and fast
- Errors occur rarely, so I/O impact is minimal
- For high-frequency errors, rate limiting prevents log file bloat

**Best practices:**
- Initialize ErrorHandler once at script start
- Don't wrap every line in try-catch
- Catch only errors you can meaningfully handle
- Let unexpected errors propagate to OnError

## Related Concepts

- docs/manuals/ahk2/how-to/retry-network-operations.md - Retry logic with logging
- docs/manuals/ahk2/how-to/circuit-breaker-pattern.md - Circuit breaker with error tracking
- docs/source-reports/ahk2/ahk2_arch_04_error.md - Complete error handling architecture

## Key Takeaway

Centralized error handling transforms scattered, inconsistent error management into a robust system. Initialize ErrorHandler at script startup, catch only errors you can handle locally, and let unexpected errors propagate to the global handler. This provides consistent logging, appropriate user feedback, and debuggability while preventing error spam and lost information.
