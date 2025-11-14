# AHK2 Research Request: Closures & Function Scope

## What I Need to Learn

I'm using fat arrows `=>` everywhere but don't fully understand closures. When do variables get captured? By reference or value? How long do they live? I'm probably creating memory leaks without knowing.

## My Current Understanding (Challenge This!)

I believe:
- Fat arrow `=>` creates a closure
- Closures capture variables by value (snapshot)
- Captured variables die when function ends
- Regular functions can't create closures
- Closures are slower than regular functions

**Correct my misconceptions.**

## The Bugs I've Created

**Bug 1: Loop closure problem**
```ahk
callbacks := []
Loop 5 {
    callbacks.Push(() => MsgBox(A_Index))
}

callbacks[1]()  // I expected 1, got 5 - WHY?
callbacks[5]()  // Also shows 5 - WHY?
```

**Bug 2: Memory leak?**
```ahk
CreateHandler() {
    largeObject := {data: MassiveArray()}
    return () => ProcessData(largeObject.data)
}

handler := CreateHandler()
// Is largeObject kept in memory forever?
```

**Bug 3: Scope confusion**
```ahk
x := 10

TestFunc() {
    y := 20
    callback := () => MsgBox(x + y)
    return callback
}

cb := TestFunc()
cb()  // Does this work? What about y?
```

## Specific Questions

1. **Capture Mechanism**: Do closures capture by reference or value? Can I force value capture?

2. **Lifetime**: How long do captured variables live? Until closure is freed?

3. **Fat Arrow vs Regular**: What's the difference in closure behavior?

4. **Performance**: What's the memory overhead of closure? Call overhead?

5. **Loop Captures**: Why does the loop problem occur? How to fix it?

## How to Write This Guide

### 1. Start with "The Closure Mental Model"

Visualize memory:
```
Code:
  CreateCounter() {
      count := 0
      return () => ++count
  }

  counter := CreateCounter()

Memory layout:
┌─────────────────┐
│ Heap Object     │
│ ┌─────────────┐ │
│ │ count: 0    │ │ ← Captured variable
│ └─────────────┘ │
└────────↑────────┘
         │
    ┌────┴─────┐
    │ counter  │ ← Closure references heap object
    │ function │
    └──────────┘
```

### 2. Capture by Reference Proof

Show it experimentally:
```ahk
TestCapture() {
    x := 10
    closure := () => MsgBox(x)

    MsgBox "Before change"
    closure()  // Shows 10

    x := 20    // Change captured variable

    MsgBox "After change"
    closure()  // Shows 20 or 10?

    // ANSWER: Shows 20 (reference, not value!)
}
```

Then explain the implications.

### 3. The Loop Problem Explained

Detailed walkthrough:
```ahk
; The problem:
closures := []
Loop 3 {
    closures.Push(() => MsgBox(A_Index))
}

// At end of loop, A_Index = 3
// All closures reference THE SAME A_Index variable
// So all show 3

closures[1]()  // 3 (not 1!)
closures[2]()  // 3 (not 2!)
closures[3]()  // 3 (not 3!)
```

Then show THREE solutions:

**Solution 1: Capture in local**
```ahk
closures := []
Loop 3 {
    local i := A_Index  // Create local copy
    closures.Push(() => MsgBox(i))
}
```

**Solution 2: Parameter binding**
```ahk
closures := []
Loop 3 {
    closures.Push(((val) => () => MsgBox(val))(A_Index))
}
```

**Solution 3: Using Bind**
```ahk
ShowIndex(i) => MsgBox(i)
closures := []
Loop 3 {
    closures.Push(ShowIndex.Bind(A_Index))
}
```

Explain which is clearest.

### 4. Closure Patterns

**Pattern 1: Counter**
```ahk
MakeCounter(start := 0) {
    count := start
    return {
        inc: () => ++count,
        dec: () => --count,
        get: () => count,
        reset: () => count := start
    }
}

c := MakeCounter(10)
c.inc()  // 11
c.inc()  // 12
c.get()  // 12
```

**Pattern 2: Private state**
```ahk
CreateDatabase() {
    data := Map()  // Private, can't access from outside

    return {
        set: (k, v) => data[k] := v,
        get: (k) => data[k],
        has: (k) => data.Has(k)
    }
    // 'data' is private - true encapsulation!
}
```

**Pattern 3: Configuration builder**
```ahk
ConfigBuilder() {
    settings := Map()

    return {
        set: (k, v) => (settings[k] := v, this),  // Return this for chaining
        get: (k, def := "") => settings.Has(k) ? settings[k] : def,
        build: () => settings.Clone()  // Return copy
    }
}

config := ConfigBuilder()
    .set("host", "localhost")
    .set("port", 8080)
    .build()
```

### 5. Memory Management

Explain closure memory:
```ahk
; Does this leak memory?
CreateLeak() {
    huge := Array(1000000)  // 1 million elements
    return () => "I don't use huge"  // But closure captures it anyway!
}

leak := CreateLeak()
// 'huge' stays in memory as long as 'leak' exists
```

Then show solutions:
- Only capture what you need
- Unset variables before returning closure
- Break circular references

### 6. Scope Rules Comparison

| Code Pattern | Regular Function | Fat Arrow | What Gets Captured |
|--------------|------------------|-----------|-------------------|
| Access global | Can read without declare | Can read without declare | Nothing |
| Modify global | Must declare global | assume-global by default | Nothing |
| Access outer local | Can read outer scope | Can read outer scope | By reference |
| Modify outer local | Can modify | Can modify | By reference |

### 7. Real-World Transforms

Take real feature from my code:

**Before (tight coupling):**
```ahk
global windowGroup := {window1: 0, window2: 0}
global windowDimmer := Map()

; WindowGrouping directly accesses dimmer
CreateGroup(w1, w2) {
    windowGroup.window1 := w1
    windowGroup.window2 := w2

    ; Direct coupling - bad!
    if (windowDimmer.Has(w1))
        windowDimmer.Delete(w1)
    if (windowDimmer.Has(w2))
        windowDimmer.Delete(w2)
}
```

**After (event-driven with closures):**
```ahk
class WindowGrouping {
    CreateGroup(w1, w2) {
        this.group := {window1: w1, window2: w2}

        ; Emit event instead of direct call
        EventBus.Emit("window:grouped", {windows: [w1, w2]})
    }
}

class WindowDimmer {
    __New() {
        ; Subscribe to events
        EventBus.On("window:grouped", (data) => {
            for hwnd in data.windows
                this.Exclude(hwnd)
        })
    }
}
```

### 8. Performance Comparison

Benchmark:
```
Test: Call 100k times

Direct function call: 8ms
Fat arrow call: 10ms  (+25%)
Closure call: 12ms  (+50%)

Memory:
Direct: 0 bytes overhead
Closure: ~40 bytes + captured variables

Conclusion: Closures have cost, but negligible unless creating thousands
```

### 9. When to Use Closures

Decision tree:
```
Need to capture outer variables?
├─ No → Use regular function
└─ Yes ↓
  Need state across calls?
  ├─ No → Consider parameters instead
  └─ Yes ↓
    Need true privacy?
    ├─ Yes → Use closure (can't access from outside)
    └─ No → Use class instance (more standard)
```

### 10. The Gotchas

**"All my closures show the same value!"**
> Loop problem - all closures capture same variable

**"Memory leak I can't find!"**
> Closure captured huge object I thought was dead

**"Timer callback can't access variable!"**
> Forgot timer runs in different scope - need closure to capture context

**"Closure updating wrong variable!"**
> Captured reference to variable that got reassigned

## Success Criteria

After this guide:
1. ✓ Understand reference vs value capture
2. ✓ Solve loop closure problem
3. ✓ Create true private state using closures
4. ✓ Prevent closure memory leaks
5. ✓ Use closures for event handlers effectively

## Most Important

I want to understand **how closures work at memory level** - what gets allocated where, what references exist, when things get freed.

Not just syntax, but the underlying mechanism that makes closures possible.
