# Explanation: How Closures Actually Work

**Understanding the memory model that makes closures powerful—and dangerous**

Closures in AutoHotkey v2 are often described as "functions that remember values," but this mental model leads to infinite confusion. The truth is more elegant and more dangerous: closures capture variables **by reference**, storing pointers to memory locations in heap-allocated "free variable sets" that outlive function returns. This architectural choice enables powerful patterns like private state and event handlers, but also creates the notorious loop closure problem and potential memory leaks.

## The Mental Model Shift

### What You Probably Think

- Closures "remember" or "capture" values
- Each closure gets its own copy of variables
- The loop problem is a bug in AHK2
- Fat arrows create closures, regular functions don't

### The Reality

- Closures capture **references** (memory addresses), not values
- All closures from one function invocation **share** the same variables
- The loop problem is by design—reference capture enables shared mutable state
- Both fat arrows and regular nested functions create identical closures

## Memory Architecture: How It Actually Works

### The Free Variable Set

When you create a closure, AHK2 doesn't copy variables. Instead, it allocates a single **free variable set** on the heap containing all captured variables. This set is shared by:

1. The outer function (while it's executing)
2. All nested functions created in that invocation
3. Any closures returned or registered as callbacks

```
┌─────────────────────────────────────────┐
│ Stack Frame: MakeCounter() invocation  │
│                                         │
│  count (local) ──────┐                 │
└──────────────────────┼──────────────────┘
                       │
                       ▼
         ┌──────────────────────────┐
         │ HEAP: Free Variable Set  │
         │ ┌──────────────────────┐ │
         │ │ count: 0             │ │
         │ └──────────────────────┘ │
         └────▲──────────▲──────────┘
              │          │
    ┌─────────┘          └─────────┐
    │                              │
┌───┴────────────┐    ┌────────────┴────┐
│ Closure A      │    │ Closure B       │
│ References     │    │ References      │
│ free var set   │    │ same free var   │
└────────────────┘    └─────────────────┘
```

**Key insight:** When you call the outer function again, you get a **completely separate** free variable set. Closures from different invocations don't share state.

### Reference Capture Proof

The definitive test shows that closures share mutable state:

```ahk
f(c) {
    g() {
        MsgBox c .= ' ' . c  ; Modifies captured variable
    }
    h() {
        MsgBox c := Chr(Ord(c)+1)  ; Also modifies same variable
    }
    Hotkey 'f1', g
    Hotkey 'f2', h
}
f('a')

; Press F1 → Shows "a a" (c modified from "a" to "a a")
; Press F2 → Shows "b" (c was "a a", increments 'a' to 'b')
; Press F1 again → Shows "b b" (c is now "b")
```

Both closures `g` and `h` share the **exact same** `c` variable. This proves capture by reference, not by value.

## The Loop Problem: Why It Happens

The classic closure bug that confuses everyone:

```ahk
callbacks := []
Loop 5 {
    callbacks.Push(() => MsgBox(A_Index))
}
callbacks[1]()  // Shows 5, not 1
callbacks[5]()  // Also shows 5
```

### Step-by-Step Memory Walkthrough

**Iteration 1:**
- `A_Index` = 1
- Create closure: `() => MsgBox(A_Index)`
- Closure stores: **reference to A_Index variable**
- Not stored: the value 1

**Iteration 2:**
- `A_Index` = 2
- Create closure: `() => MsgBox(A_Index)`
- Closure stores: **reference to same A_Index variable**
- Both closures now point to the same memory location

**Iterations 3-5:**
- Same pattern: all closures capture **the same variable**

**After loop completes:**
- `A_Index` = 5 (final loop value)
- All 5 closures reference **that same variable**
- When you call any closure, it evaluates `A_Index` and finds 5

### Why This Design?

Reference capture enables:

1. **Shared mutable state** - Multiple closures can coordinate
2. **Private state** - Variables outlive function return
3. **Reactive patterns** - Closures see updates from other closures

The loop problem is the price of this power. You're not meant to share loop variables across closures.

## Solution Patterns

### Pattern 1: Local Variable Capture (Recommended)

Create a new variable inside the loop body:

```ahk
callbacks := []
Loop 5 {
    i := A_Index  ; New variable, unique memory location
    callbacks.Push(() => MsgBox(i))
}
callbacks[1]()  ; Shows 1 ✓
callbacks[5]()  ; Shows 5 ✓
```

**Why this works:** Each iteration creates a **different variable** `i` at a different memory address. The first closure captures the first `i`, the second captures the second `i`, etc.

### Pattern 2: Default Parameter Binding

Default parameters evaluate at closure creation time:

```ahk
callbacks := []
Loop 5 {
    callbacks.Push((idx := A_Index) => MsgBox(idx))
}
callbacks[1]()  ; Shows 1 ✓
```

**Why this works:** `idx := A_Index` executes when the closure is created, binding the current value. The parameter `idx` becomes a local variable inside the closure.

**Caveat:** Less readable for those unfamiliar with the pattern.

### Pattern 3: The Bind Method

Create a BoundFunc that binds parameter values:

```ahk
callbacks := []
Loop 5 {
    callbacks.Push(((i, *) => MsgBox(i)).Bind(A_Index))
}
callbacks[1]()  ; Shows 1 ✓
```

**Why this works:** `.Bind()` stores **values**, not references. It evaluates `A_Index` immediately and stores that value in the BoundFunc object.

## Practical Closure Patterns

### Counter Pattern: Persistent Private State

```ahk
MakeCounter() {
    count := 0  ; Captured by closure
    increment() {
        count += 1
        return count
    }
    return increment
}

counter1 := MakeCounter()
MsgBox counter1()  ; 1
MsgBox counter1()  ; 2
MsgBox counter1()  ; 3

counter2 := MakeCounter()  ; Independent state
MsgBox counter2()  ; 1
MsgBox counter1()  ; 4
```

**What's happening:**
- Each `MakeCounter()` call creates a separate free variable set
- `counter1` and `counter2` have completely independent `count` variables
- The `count` variable persists as long as the closure exists

### Observer Pattern: Multiple Closures, Shared State

```ahk
MakeAccount(initialBalance) {
    balance := initialBalance  ; Shared by all methods

    deposit(amount) {
        balance += amount
        return balance
    }

    withdraw(amount) {
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

account := MakeAccount(100)
MsgBox account.deposit(50)   ; 150
MsgBox account.withdraw(30)  ; 120
MsgBox account.getBalance()  ; 120
```

**Key insight:** All three closures share the **same** `balance` variable. Changes from `deposit` are visible to `withdraw` because they reference the same memory location.

### GUI Event Handlers: Eliminating Globals

**Before (global pollution):**

```ahk
global count := 0
global counterText := ""

IncrementCount(*) {
    global count, counterText
    count++
    counterText.Text := "Count: " count
}
```

**After (closures):**

```ahk
CreateCounterGUI() {
    count := 0  ; Captured by closure, no global

    myGui := Gui()
    counterText := myGui.Add("Text", "w200", "Count: 0")

    myGui.Add("Button", "", "Increment").OnEvent("Click", (*) => (
        count++,
        counterText.Text := "Count: " count
    ))

    myGui.Add("Button", "", "Reset").OnEvent("Click", (*) => (
        count := 0,
        counterText.Text := "Count: 0"
    ))

    myGui.Show()
}
```

Both button handlers share `count` and `counterText` through closure capture. No globals needed.

## Memory Management and Leaks

### Lifetime Extension

Normal local variables are freed when the function returns. Captured variables **persist indefinitely** until all closures referencing them are released:

```ahk
CreateHandler() {
    largeObject := {data: MassiveArray()}  ; Large allocation
    return () => ProcessData(largeObject.data)
}

handler := CreateHandler()
// largeObject still exists in memory because closure references it
```

**The leak:** Even though you only need `largeObject.data`, the entire object stays in memory because the closure references it.

**Solution:** Extract only needed data:

```ahk
CreateHandler() {
    largeObject := {data: MassiveArray()}
    dataReference := largeObject.data  ; Extract specific data
    return () => ProcessData(dataReference)
}
// Now largeObject can be garbage collected
```

### Circular References

Closures storing themselves in captured variables create reference cycles:

```ahk
BadPattern() {
    callbacks := []  ; Will be captured

    fn := () => callbacks.Push("new")  ; Captures callbacks
    callbacks.Push(fn)  ; Store closure in captured array

    return callbacks  ; Circular reference
}
```

The closure captures `callbacks`, and `callbacks` contains the closure. This circular reference can delay garbage collection.

**Avoiding cycles:** Don't store closures in the variables they capture.

## Scope Rules: Regular vs Fat Arrow Functions

**Myth:** Fat arrows create closures, regular functions don't.

**Reality:** Both create identical closures when nested and referencing outer variables.

```ahk
TestModification() {
    outerVar := 10

    // Fat arrow modifies captured variable
    fatArrowFn := () => outerVar := 20
    fatArrowFn()
    MsgBox outerVar  ; 20

    outerVar := 10

    // Regular function also modifies captured variable
    RegularFn() {
        outerVar := 30
    }
    RegularFn()
    MsgBox outerVar  ; 30
}
```

Both modify `outerVar` through the captured reference. The **only** difference is syntax.

### Key Differences

| Feature | Fat Arrow | Regular Function |
|---------|-----------|------------------|
| Create closures? | Yes (when nested) | Yes (when nested) |
| Capture by reference? | Yes | Yes |
| Can modify captured vars? | Yes | Yes |
| Multi-statement body? | v2.1+ only | Always |
| Explicit declarations? | No | Yes (local, static, global) |

The nesting context matters more than function type.

## Performance Considerations

**Community consensus:** Closure overhead is negligible for typical AutoHotkey usage.

**Estimated costs:**
- Memory: 200-500 bytes per closure object
- Call overhead: Minimal indirection through closure object
- Captured variables: Shared across closures (memory-efficient)

**When overhead matters:**
- Tight loops creating thousands of closures per second
- Memory-constrained environments
- High-frequency callbacks (>10,000/second)

**When it doesn't matter:**
- GUI event handlers
- Hotkey callbacks
- Timers running every few seconds
- Typical automation scripts

For 99% of use cases: **just use closures**. The code clarity benefits far outweigh unmeasurable performance costs.

## Common Misconceptions Corrected

### Misconception 1: "Fat arrows create closures, regular functions don't"

**Reality:** Both create closures identically when nested and referencing outer variables. Closure creation depends on variable references, not syntax.

### Misconception 2: "Closures capture by value"

**Reality:** Closures capture by reference, creating shared mutable state. All closures from one invocation share the same variables.

### Misconception 3: "Captured variables die when the outer function ends"

**Reality:** Captured variables extend their lifetime until all closures referencing them are freed. They're heap-allocated and persist beyond function return.

### Misconception 4: "The loop problem is a bug"

**Reality:** It's by design. Reference capture enables closures to maintain shared state and modify outer variables. The solution is creating separate variables, not "fixing" AHK.

### Misconception 5: "Closures are significantly slower"

**Reality:** Call overhead is negligible for typical usage. Community consensus: "just use them wherever possible" without micro-optimization concerns.

## The Core Principle

Understanding closures requires shifting from "functions that remember values" to "functions that share variables through memory references."

**The mental model:**
- When a closure is created, it doesn't copy the world
- It joins a shared context where multiple functions point to the same memory locations
- This explains: why loop closures fail, why modifications propagate, why memory stays allocated, why the solution involves creating new variables

**When to use closures:**
- GUI event handlers (primary use case)
- Private state encapsulation
- Callbacks to built-in functions (SetTimer, Hotkey)
- Observer/reactive patterns
- Eliminating global variables

**When to avoid closures:**
- Simple parameter binding (use `.Bind()` instead)
- No captured state needed
- Extreme performance scenarios (rare)

Master reference capture, and closures become one of your most powerful tools for clean, maintainable AutoHotkey v2 code.

## Cross-References

Related topics:
- docs/manuals/ahk2/reference/function-objects.md - Function object types
- docs/manuals/ahk2/how-to/event-handlers.md - GUI event handling patterns
- docs/manuals/ahk2/explanation/class-system.md - Understanding scope and lifetime
