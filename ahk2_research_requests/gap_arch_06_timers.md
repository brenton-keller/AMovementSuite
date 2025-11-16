# AHK2 Research Request: Timers & Async Patterns

## What I Need to Learn

I use SetTimer but don't understand execution model. Can timers interrupt each other? Are they async? How do I prevent race conditions? I need deep understanding of how timers work.

## My Current Understanding (Challenge This!)

- Timers run in parallel (async/threaded)
- Timers can interrupt any code at any time
- SetTimer with negative period runs once
- You can have unlimited timers running

**Correct me.**

## The Problems

**Problem 1: Race condition?**
```ahk
global updateInProgress := false

UpdateWindow() {
    if (updateInProgress)
        return  // Already running

    updateInProgress := true
    WinMove ...
    updateInProgress := false
}

SetTimer UpdateWindow, 5  // Run every 5ms
// Can two calls run simultaneously? Race condition?
```

**Problem 2: Can't catch errors**
```ahk
try {
    SetTimer () => ThrowError(), -100
} catch {
    // Never caught - why?
}
```

**Problem 3: Timer cleanup**
```ahk
// I have 15 timers running
// How do I stop them all?
// How do I prevent duplicate timers?
```

## Specific Questions

1. **Execution Model**: Is AHK2 single-threaded? How do timers work if so?

2. **Interruption**: Can timer fire while other code running? What happens?

3. **Timer Accuracy**: I set 5ms timer - does it fire exactly every 5ms?

4. **Error Handling**: How do I handle errors in timer callbacks?

5. **Cleanup**: How do I track and stop all timers? Prevent leaks?

## How to Write This Guide

### 1. Explain Single-Threaded Model

```
AHK2 is SINGLE-THREADED

But timers seem async? How?

Answer: Timer doesn't interrupt - it QUEUES
```

Show diagram:
```
Timeline:
[Main Code Running........................]
         ↑
         Timer fires (time to run callback)
         ↓
       [Queued] - waits for main code to finish
                     ↓
                   Main code finishes
                     ↓
                   [Timer callback runs]
                     ↓
                   [Main code resumes]
```

### 2. Critical vs Non-Critical Timers

Explain difference:
```ahk
SetTimer Func, 100  // Non-critical (skips if previous call still running)
SetTimer Func, 100, 1  // Critical (queues, all calls execute)

// When to use each?
```

### 3. Timer Patterns

**Debounce** (wait for activity to stop):
```ahk
Debounce(func, delay := 500) {
    static timers := Map()
    key := func.Name

    ; Cancel existing timer
    if timers.Has(key)
        SetTimer timers[key], 0

    ; Create new timer
    timer := () => (func(), timers.Delete(key))
    timers[key] := timer
    SetTimer timer, -delay
}

// Usage: Only execute after user stops typing for 500ms
Debounce(SaveDocument, 500)
```

**Throttle** (limit execution rate):
```ahk
Throttle(func, interval := 100) {
    static lastCall := Map()
    key := func.Name

    now := A_TickCount
    if lastCall.Has(key) && (now - lastCall[key]) < interval
        return false  // Too soon

    lastCall[key] := now
    func()
    return true
}

// Usage: Execute at most once per 100ms
```

**Timeout**:
```ahk
Timeout(func, timeoutMs, onTimeout) {
    completed := false

    ; Run function
    funcTimer := () => (func(), completed := true)
    SetTimer funcTimer, -1

    ; Set timeout
    timeoutTimer := () => !completed && onTimeout()
    SetTimer timeoutTimer, -timeoutMs
}
```

### 4. Real-World Transform

My actual code:
```ahk
; Window scaling with live updates
SetTimer UpdateActualWindow_Width, 5  // Every 5ms

UpdateActualWindow_Width() {
    // 15 global variables
    // Manual state tracking
    // Race condition potential
}

// Later:
SetTimer UpdateActualWindow_Width, 0  // Stop
```

Transform to:
```ahk
class WindowScaler {
    static instances := Map()

    StartScaling(hwnd) {
        scaler := WindowScaler(hwnd)
        this.instances[hwnd] := scaler
        scaler.timer := () => scaler.Update()
        SetTimer scaler.timer, 5
    }

    StopScaling(hwnd) {
        if !this.instances.Has(hwnd)
            return

        scaler := this.instances[hwnd]
        SetTimer scaler.timer, 0
        this.instances.Delete(hwnd)
    }

    Update() {
        // Instance variables, no globals
        // No race conditions
    }
}
```

### 5. Error Handling in Timers

The problem:
```ahk
SetTimer () => ThrowError(), -100
// Error occurs but can't be caught from outside
```

The solutions:
```ahk
// Solution 1: Handle inside timer
SafeTimer() {
    try {
        RiskyOperation()
    } catch Error as e {
        LogError(e)
    }
}
SetTimer SafeTimer, -100

// Solution 2: Wrapper
class SafeTimer {
    static Start(callback, period, errorHandler := "") {
        wrapper := () {
            try callback()
            catch Error as e {
                if errorHandler
                    errorHandler(e)
                else
                    LogError(e)
            }
        }
        SetTimer wrapper, period
    }
}
```

### 6. Performance & Accuracy

Test timer accuracy:
```ahk
; Test: 100ms timer actual frequency
count := 0
start := A_TickCount

TestTimer() {
    global count
    count++
}

SetTimer TestTimer, 100
Sleep 10000
SetTimer TestTimer, 0

actualInterval := 10000 / count
MsgBox "Requested 100ms, actual " actualInterval "ms"
```

What affects accuracy? System load? Timer period?

### 7. The Gotchas

**"Timer runs forever!"**
> Forgot to stop timer - memory leak from accumulating state

**"Timer skipped executions!"**
> Used non-critical timer, previous call still running

**"Race condition!"**
> TWO different timers accessed same global state

**"Can't stop timer!"**
> Used anonymous function, no reference to pass to SetTimer func, 0

## Success Criteria

1. ✓ Understand single-threaded execution model
2. ✓ Implement debounce and throttle
3. ✓ Handle errors in timer callbacks
4. ✓ Prevent timer memory leaks
5. ✓ Choose critical vs non-critical correctly

## Most Important

I want to understand **how timers work** at execution level - the message queue, thread model, and why certain bugs occur.

Teach me to think about timers as "scheduled code execution in single-threaded environment", not "async parallel tasks."
