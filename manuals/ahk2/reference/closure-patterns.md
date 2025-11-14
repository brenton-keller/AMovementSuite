# Reference: Closure Patterns

Catalog of closure patterns with use cases, implementation details, and performance characteristics.

## Related Documentation
- docs/manuals/ahk2/reference/class-system.md - Alternative to closures for state management
- docs/manuals/ahk2/reference/property-system.md - Properties with closures

---

## Closure Fundamentals

**Key Concept:** Closures capture variables by **reference**, not by value.

**Memory Model:** Captured variables live in heap-allocated "free variable sets" shared by all closures from one outer function invocation.

**Lifetime:** Captured variables persist until all closures referencing them are freed.

---

## Pattern Catalog

### 1. Counter (Private State)

**Purpose:** Maintain private incrementing counter.

**Implementation:**

```ahk
MakeCounter(start := 0) {
    count := start  ; Captured by closure

    increment() {
        count++
        return count
    }

    return increment
}

; Usage
counter1 := MakeCounter()
MsgBox counter1()  ; 1
MsgBox counter1()  ; 2
MsgBox counter1()  ; 3

counter2 := MakeCounter(100)
MsgBox counter2()  ; 101
MsgBox counter2()  ; 102
```

**Characteristics:**
- Each invocation creates independent counter
- State persists across calls
- No way to access `count` except through closure
- True encapsulation

**Use When:** Need simple incrementing counter, ID generator, sequence generator.

---

### 2. Private State Object (Multiple Methods)

**Purpose:** Multiple functions sharing private state.

**Implementation:**

```ahk
MakeAccount(initialBalance) {
    balance := initialBalance  ; Shared private state

    deposit(amount) {
        if (amount <= 0)
            throw ValueError("Amount must be positive")
        balance += amount
        return balance
    }

    withdraw(amount) {
        if (amount <= 0)
            throw ValueError("Amount must be positive")
        if (amount > balance)
            throw ValueError("Insufficient funds")
        balance -= amount
        return balance
    }

    getBalance() {
        return balance
    }

    return {
        deposit: deposit,
        withdraw: withdraw,
        getBalance: getBalance
    }
}

; Usage
account := MakeAccount(100)
account.deposit(50)       ; 150
account.withdraw(30)      ; 120
MsgBox account.getBalance()  ; 120
```

**Characteristics:**
- Multiple closures share same state
- State is completely private
- Object-like interface without classes
- All methods can modify shared state

**Use When:** Need encapsulation without classes, simple state management, module pattern.

---

### 3. Configuration Builder (Validated State)

**Purpose:** Encapsulated configuration with validation.

**Implementation:**

```ahk
CreateSettings(defaultTimeout := 5000) {
    timeout := defaultTimeout
    maxRetries := 3
    isEnabled := true

    return {
        GetTimeout: () => timeout,

        SetTimeout: (value) => (
            timeout := Max(100, Min(value, 60000)),
            timeout
        ),

        GetRetries: () => maxRetries,

        SetRetries: (value) => (
            maxRetries := Max(1, Min(value, 10)),
            maxRetries
        ),

        Toggle: () => (isEnabled := !isEnabled),
        IsEnabled: () => isEnabled
    }
}

; Usage
settings := CreateSettings()
settings.SetTimeout(10000)
MsgBox settings.GetTimeout()  ; 10000
settings.SetTimeout(100000)   ; Clamped to 60000
MsgBox settings.GetTimeout()  ; 60000
```

**Characteristics:**
- Validation built into setters
- Getters/setters as closures
- Constraints enforced automatically
- Private state with public interface

**Use When:** Need validated configuration, bounded values, settings management.

---

### 4. Event Handler with Context

**Purpose:** GUI event handlers with captured context.

**Implementation:**

```ahk
CreateCounterGUI() {
    count := 0  ; Captured by both handlers

    myGui := Gui(, "Counter Demo")
    counterText := myGui.Add("Text", "w200", "Count: 0")

    myGui.Add("Button", "", "Increment").OnEvent("Click", (*) => (
        count++,
        counterText.Text := "Count: " . count
    ))

    myGui.Add("Button", "", "Reset").OnEvent("Click", (*) => (
        count := 0,
        counterText.Text := "Count: 0"
    ))

    myGui.Add("Button", "", "Double").OnEvent("Click", (*) => (
        count *= 2,
        counterText.Text := "Count: " . count
    ))

    myGui.Show()
}

; Usage
CreateCounterGUI()
```

**Characteristics:**
- No global variables needed
- All handlers share state naturally
- Context stays with GUI creation
- Clean, readable event handlers

**Use When:** GUI event handlers, timer callbacks, hotkey handlers.

---

### 5. Callback Factory (Parameterized Closures)

**Purpose:** Create multiple callbacks with different parameters.

**Implementation:**

```ahk
MakeToggleHandler(name, menu) {
    state := true

    handler(*) {
        state := !state
        if state
            menu.Check(name)
        else
            menu.Uncheck(name)
    }

    return handler
}

; Usage
myMenu := Menu()
myMenu.Add("Auto-Save", MakeToggleHandler("Auto-Save", myMenu))
myMenu.Add("Spell Check", MakeToggleHandler("Spell Check", myMenu))
myMenu.Add("Dark Mode", MakeToggleHandler("Dark Mode", myMenu))
myMenu.Show()
```

**Characteristics:**
- Each closure captures different parameters
- Independent state per handler
- Factory pattern for callbacks
- Scalable to many instances

**Use When:** Multiple similar callbacks, menu items, toolbar buttons.

---

### 6. Lazy Evaluation (Deferred Computation)

**Purpose:** Compute expensive value only when first needed.

**Implementation:**

```ahk
MakeLazyValue(computeFunc) {
    computed := false
    cachedValue := ""

    getValue() {
        if (!computed) {
            cachedValue := computeFunc()
            computed := true
        }
        return cachedValue
    }

    return getValue
}

; Usage
expensiveCalc() {
    Sleep(1000)  ; Simulate expensive operation
    return "Computed result"
}

lazyValue := MakeLazyValue(expensiveCalc)

; First access - slow (computes)
MsgBox lazyValue()  ; Takes 1 second

; Subsequent access - instant (cached)
MsgBox lazyValue()  ; Instant
MsgBox lazyValue()  ; Instant
```

**Characteristics:**
- Computation deferred until needed
- Result cached after first computation
- No re-computation on subsequent calls
- Memory efficient (only allocates when needed)

**Use When:** Expensive computations, resource loading, initialization that might not be needed.

---

### 7. Memoization (Cache Function Results)

**Purpose:** Cache function results based on arguments.

**Implementation:**

```ahk
Memoize(func) {
    cache := Map()

    memoized(args*) {
        ; Create cache key from arguments
        key := ""
        for arg in args
            key .= String(arg) . "|"

        ; Return cached if exists
        if cache.Has(key)
            return cache[key]

        ; Compute and cache
        result := func(args*)
        cache[key] := result
        return result
    }

    return memoized
}

; Usage - expensive recursive function
Fibonacci(n) {
    if (n <= 1)
        return n
    return Fibonacci(n-1) + Fibonacci(n-2)
}

FastFib := Memoize(Fibonacci)
MsgBox FastFib(40)  ; Much faster than non-memoized
```

**Characteristics:**
- Caches results for each unique argument set
- Dramatically speeds up repeated calls
- Memory trade-off (stores all results)
- Best for pure functions (deterministic)

**Use When:** Expensive pure functions, recursive algorithms, repeated calculations.

---

### 8. Partial Application (Pre-fill Parameters)

**Purpose:** Create specialized functions by pre-filling parameters.

**Implementation:**

```ahk
Partial(func, boundArgs*) {
    partial(additionalArgs*) {
        allArgs := []
        for arg in boundArgs
            allArgs.Push(arg)
        for arg in additionalArgs
            allArgs.Push(arg)
        return func(allArgs*)
    }
    return partial
}

; Usage
Add(x, y) => x + y

Add5 := Partial(Add, 5)
MsgBox Add5(10)  ; 15
MsgBox Add5(20)  ; 25

Multiply(x, y, z) => x * y * z
Double := Partial(Multiply, 2)
MsgBox Double(3, 4)  ; 24 (2 * 3 * 4)
```

**Characteristics:**
- Creates specialized functions from general ones
- Captures initial arguments
- Remaining arguments provided at call time
- Functional programming pattern

**Use When:** Need multiple variations of a function, callback adapters, API wrappers.

**Note:** AHK2's built-in `.Bind()` provides similar functionality but binds parameters rather than capturing them.

---

### 9. Loop Closure (Correct Pattern)

**Problem:** Loop variables captured by reference cause all closures to share final value.

**Wrong Pattern:**

```ahk
callbacks := []
Loop 5 {
    callbacks.Push(() => MsgBox(A_Index))
}
callbacks[1]()  ; Shows 5 (not 1!)
```

**Correct Pattern 1 - Local Variable:**

```ahk
callbacks := []
Loop 5 {
    i := A_Index  ; Create unique variable per iteration
    callbacks.Push(() => MsgBox(i))
}
callbacks[1]()  ; Shows 1 ✓
callbacks[5]()  ; Shows 5 ✓
```

**Correct Pattern 2 - Default Parameter:**

```ahk
callbacks := []
Loop 5 {
    callbacks.Push((idx := A_Index) => MsgBox(idx))
}
callbacks[1]()  ; Shows 1 ✓
```

**Correct Pattern 3 - Bind Method:**

```ahk
callbacks := []
Loop 5 {
    callbacks.Push(((i, *) => MsgBox(i)).Bind(A_Index))
}
callbacks[1]()  ; Shows 1 ✓
```

**Characteristics:**
- Local variable: Most readable, recommended
- Default parameter: Most compact
- Bind: Most explicit about value capture

**Use When:** Creating closures in loops (very common).

---

### 10. Resource Manager (Cleanup Pattern)

**Purpose:** Automatic resource cleanup with closures.

**Implementation:**

```ahk
WithFile(path, callback) {
    file := FileOpen(path, "r")
    try {
        return callback(file)
    } finally {
        file.Close()  ; Always closes
    }
}

; Usage
content := WithFile("data.txt", (f) => f.Read())
```

**Characteristics:**
- Guarantees cleanup with finally
- Resource lifetime scoped to callback
- Prevents resource leaks
- Similar to Python's "with" statement

**Use When:** File operations, database connections, API sessions.

---

### 11. State Machine (Closure-Based)

**Purpose:** Implement state machine without classes.

**Implementation:**

```ahk
MakeStateMachine(initialState) {
    currentState := initialState
    transitions := Map()

    addTransition(from, event, to) {
        key := from . "|" . event
        transitions[key] := to
    }

    trigger(event) {
        key := currentState . "|" . event
        if transitions.Has(key) {
            currentState := transitions[key]
            return true
        }
        return false
    }

    getState() => currentState

    return {
        addTransition: addTransition,
        trigger: trigger,
        getState: getState
    }
}

; Usage
fsm := MakeStateMachine("idle")
fsm.addTransition("idle", "start", "running")
fsm.addTransition("running", "pause", "paused")
fsm.addTransition("paused", "resume", "running")
fsm.addTransition("running", "stop", "idle")

MsgBox fsm.getState()  ; "idle"
fsm.trigger("start")
MsgBox fsm.getState()  ; "running"
fsm.trigger("pause")
MsgBox fsm.getState()  ; "paused"
```

**Characteristics:**
- Encapsulated state and transitions
- Simple state machine without OOP
- Easy to reason about
- Good for simple workflows

**Use When:** Simple state management, workflow control, mode switching.

---

### 12. Observer Pattern (Publish-Subscribe)

**Purpose:** Implement event system with closures.

**Implementation:**

```ahk
MakeEventEmitter() {
    listeners := Map()

    on(event, callback) {
        if !listeners.Has(event)
            listeners[event] := []
        listeners[event].Push(callback)
    }

    off(event, callback) {
        if !listeners.Has(event)
            return
        arr := listeners[event]
        for i, cb in arr {
            if (cb = callback) {
                arr.RemoveAt(i)
                break
            }
        }
    }

    emit(event, data?) {
        if !listeners.Has(event)
            return
        for callback in listeners[event] {
            try {
                IsSet(data) ? callback(data) : callback()
            } catch as e {
                OutputDebug("Listener error: " . e.Message)
            }
        }
    }

    return {on: on, off: off, emit: emit}
}

; Usage
emitter := MakeEventEmitter()
emitter.on("data", (value) => MsgBox("Received: " . value))
emitter.on("data", (value) => FileAppend(value . "`n", "log.txt"))
emitter.emit("data", "Hello")
```

**Characteristics:**
- Decouples publisher from subscribers
- Multiple listeners per event
- Error isolation (one listener error doesn't break others)
- Event-driven architecture

**Use When:** Event systems, plugin architecture, loosely coupled components.

---

## Performance Characteristics

### Memory Overhead

| Pattern | Per Closure | Notes |
|---------|-------------|-------|
| Simple counter | ~200-500 bytes | Minimal |
| Multiple methods | ~200-500 bytes each | Share free variable set |
| Memoization | Variable | Grows with cache size |
| Loop closures | ~200-500 bytes each | One per iteration |

**Key Insight:** All closures from one outer function invocation share the same free variable set - memory efficient.

### Call Overhead

- **Closure call vs direct function:** Negligible (<1 microsecond difference)
- **Only matters:** Tight loops >10,000 iterations/second
- **Typical GUI/automation:** Completely irrelevant

### When Performance Matters

**Avoid closures in:**
- Tight loops processing thousands of items/second
- Real-time game engines
- High-frequency timers (<10ms intervals)

**Closures are fine for:**
- GUI event handlers
- Timer callbacks (>50ms intervals)
- Hotkey handlers
- Configuration management
- 99% of automation scripts

---

## Common Mistakes

### 1. Loop Closure Problem

```ahk
; WRONG - all closures capture same variable
Loop 5
    callbacks.Push(() => MsgBox(A_Index))

; RIGHT - create local variable
Loop 5 {
    i := A_Index
    callbacks.Push(() => MsgBox(i))
}
```

### 2. Assuming Value Capture

```ahk
; WRONG - modifying captured variable affects all closures
x := 10
fn1 := () => MsgBox(x)
fn2 := () => (x := 20)
fn1()  ; 10
fn2()  ; Changes x
fn1()  ; 20 (sees modification)

; RIGHT - use .Bind() for value capture
fn1 := ((val) => MsgBox(val)).Bind(x)
```

### 3. Capturing Large Objects Unintentionally

```ahk
; WRONG - entire object stays in memory
ProcessData(largeObject) {
    handler := () => UseData(largeObject.data)
    RegisterHandler(handler)
}

; RIGHT - extract only needed data
ProcessData(largeObject) {
    data := largeObject.data
    handler := () => UseData(data)
    RegisterHandler(handler)
}
```

### 4. Circular References

```ahk
; WRONG - circular reference leak
MakeWidget() {
    callbacks := []
    fn := () => callbacks.Push("new")
    callbacks.Push(fn)  ; fn captures callbacks which contains fn
    return callbacks
}

; RIGHT - avoid storing closures in captured variables
MakeWidget() {
    callbacks := []
    fn := () => ProcessData()  ; Doesn't reference callbacks
    callbacks.Push(fn)
    return callbacks
}
```

---

## Pattern Selection Guide

| Need | Pattern | Why |
|------|---------|-----|
| Simple counter | Counter | Minimal, straightforward |
| Multiple related functions | Private State Object | Natural grouping |
| Validated settings | Configuration Builder | Built-in validation |
| GUI handlers | Event Handler with Context | Eliminates globals |
| Many similar callbacks | Callback Factory | Scalable pattern |
| Expensive computation | Lazy Evaluation | Defers cost |
| Repeated calculations | Memoization | Caches results |
| Function specialization | Partial Application | Reusable variations |
| Loops creating callbacks | Loop Closure (Local Var) | Most readable |
| Resource cleanup | Resource Manager | Guarantees cleanup |
| State tracking | State Machine | Clear transitions |
| Event system | Observer | Decoupled communication |

---

## Closure vs Alternatives

### Closure vs Class

| Aspect | Closure | Class |
|--------|---------|-------|
| **Syntax** | Simpler for small cases | More structure |
| **Inheritance** | Not available | Full inheritance |
| **Multiple instances** | Factory function | `new ClassName()` |
| **Private data** | Truly private | Convention only |
| **Performance** | Negligible difference | Negligible difference |
| **Use when** | Simple state, few methods | Complex hierarchy, many methods |

### Closure vs Global Variable

**Always prefer closures over globals** for:
- State isolation
- Namespace cleanliness
- Testability
- Multiple independent instances

**Globals acceptable only for:**
- True application-wide constants
- Single-instance managers (already initialized)

---

## Testing Closures

### Unit Testing Pattern

```ahk
TestCounter() {
    counter := MakeCounter(0)

    ; Test increment
    if (counter() != 1)
        throw Error("Expected 1")
    if (counter() != 2)
        throw Error("Expected 2")

    ; Test independence
    counter2 := MakeCounter(100)
    if (counter2() != 101)
        throw Error("Expected 101")
    if (counter() != 3)
        throw Error("Expected 3 (not affected by counter2)")

    MsgBox "All tests passed"
}

TestCounter()
```

---

## See Also

- docs/manuals/ahk2/reference/class-system.md - Class-based alternatives
- docs/manuals/ahk2/reference/property-system.md - Properties with closures
- Official: [Functions](https://www.autohotkey.com/docs/v2/Functions.htm)
- Official: [Fat Arrow Functions](https://www.autohotkey.com/docs/v2/Language.htm#fat-arrow)
