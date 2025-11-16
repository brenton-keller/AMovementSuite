# How-To: Solve the Loop Closure Problem

## Problem Statement

When creating multiple closures inside a loop that reference the loop variable, all closures capture the same variable by reference. By the time any closure executes, the loop has finished and the variable contains its final value, causing all closures to behave identically instead of capturing their unique loop values.

**Symptom:** All callbacks show the final loop value instead of their individual iteration values.

```ahk
; BROKEN - All callbacks show 5
callbacks := []
Loop 5 {
    callbacks.Push(() => MsgBox(A_Index))
}
callbacks[1]()  ; Shows 5, not 1
callbacks[5]()  ; Also shows 5
```

## Why This Happens

AutoHotkey v2 closures **capture variables by reference, not by value**. When a nested function references an outer variable, it stores a pointer to that variable's location in memory rather than taking a snapshot of its value.

**Memory walkthrough:**
- Iteration 1: Creates closure storing reference to `A_Index` (currently 1)
- Iteration 2: Creates another closure storing reference to the **same** `A_Index` variable
- Iterations 3-5: Same pattern continues
- Loop ends: `A_Index` equals 5
- All five closures point to the same `A_Index` variable containing 5

This is by design - reference capture enables closures to maintain shared state and modify outer variables, which is essential for many patterns.

## Solution 1: Local Variable Capture (Recommended)

Create a new local variable inside each loop iteration with its own memory location:

```ahk
callbacks := []
Loop 5 {
    i := A_Index  ; New variable with unique memory location
    callbacks.Push(() => MsgBox(i))
}
callbacks[1]()  ; Shows 1 ✓
callbacks[5]()  ; Shows 5 ✓
```

**How it works:** Each iteration allocates a new `i` variable at a different memory address. The first iteration creates `i` at address X containing 1, captured by the first closure. The second iteration creates a completely different `i` at address Y containing 2, captured by the second closure. Since each closure references a different variable, they retain different values.

**Why this works:** Loop bodies in AHK2 create new scope for local variables declared within them.

**When to use:** This is the most idiomatic and recommended approach. It's immediately clear what's happening, requires minimal syntax, and works in all contexts.

## Solution 2: Default Parameter Binding

Default parameters are evaluated when the closure is created, not when it's called:

```ahk
callbacks := []
Loop 5 {
    callbacks.Push((idx := A_Index) => MsgBox(idx))
}
callbacks[1]()  ; Shows 1 ✓
```

**How it works:** The expression `idx := A_Index` executes at closure creation time. It evaluates `A_Index` (getting the current value) and stores that value in the parameter. The parameter `idx` becomes a local variable inside the closure with the captured value.

**When to use:** When you need compact syntax and your team is familiar with default parameter evaluation timing. Can confuse readers unfamiliar with this pattern.

## Solution 3: The Bind Method

The `.Bind()` method creates a BoundFunc object that binds parameter values at creation time:

```ahk
callbacks := []
Loop 5 {
    callbacks.Push(((i, *) => MsgBox(i)).Bind(A_Index))
}
callbacks[1]()  ; Shows 1 ✓
```

**How it works:** Unlike closure capture, `.Bind()` stores values, not references. When you call `.Bind(A_Index)` during iteration 3, it evaluates `A_Index` (getting 3) and stores that value in the BoundFunc object. Later invocations pass this stored value as the first parameter.

**When to use:**
- Working with existing named functions that you can't modify
- Want to make parameter binding explicit in your API design
- Need to bind parameters to functions from external libraries

**Comparison with named functions:**

```ahk
; Define function once outside loop
ShowValue(val) {
    MsgBox val
}

; Bind different values
callbacks := []
Loop 5 {
    callbacks.Push(ShowValue.Bind(A_Index))
}
```

## Quick Comparison Table

| Solution | Clarity | Flexibility | Syntax Length |
|----------|---------|-------------|---------------|
| **Local variable** | Highest | High | Short |
| **Default params** | Medium | Medium | Very short |
| **Bind method** | High | Highest | Medium |

## Complete Working Example

```ahk
#Requires AutoHotkey v2.0

; Create GUI with buttons demonstrating each solution
MyGui := Gui()
MyGui.Add("Text", , "Each button shows its number:")

; Solution 1: Local variable (recommended)
Loop 3 {
    i := A_Index
    MyGui.Add("Button", "w150", "Local Var " i).OnEvent("Click", (*) => MsgBox(i))
}

; Solution 2: Default parameter
Loop 3 {
    MyGui.Add("Button", "w150", "Default Param " A_Index).OnEvent("Click", (idx := A_Index, *) => MsgBox(idx))
}

; Solution 3: Bind method
ShowNumber(num, *) {
    MsgBox num
}
Loop 3 {
    MyGui.Add("Button", "w150", "Bind Method " A_Index).OnEvent("Click", ShowNumber.Bind(A_Index))
}

MyGui.Show()
```

## Troubleshooting

### All closures still show the same value

**Cause:** You're not creating a new variable each iteration.

```ahk
; WRONG - Still broken
i := 0
Loop 5 {
    i++  ; Modifying same variable
    callbacks.Push(() => MsgBox(i))
}
```

**Fix:** Declare new variable inside loop body:

```ahk
; CORRECT
Loop 5 {
    i := A_Index  ; New variable each iteration
    callbacks.Push(() => MsgBox(i))
}
```

### Closure captures wrong value with nested loops

**Cause:** Both loop variables need their own local copies.

```ahk
; WRONG - outer captures correctly, inner doesn't
Loop 3 {
    outer := A_Index
    Loop 3 {
        callbacks.Push(() => MsgBox(outer " - " A_Index))  ; Inner A_Index problem
    }
}
```

**Fix:** Create local for both loops:

```ahk
; CORRECT
Loop 3 {
    outer := A_Index
    Loop 3 {
        inner := A_Index
        callbacks.Push(() => MsgBox(outer " - " inner))
    }
}
```

### Bind method not working with GUI events

**Cause:** GUI event handlers receive control object and info as parameters.

```ahk
; WRONG - num gets overwritten by GUI params
ShowNum(num) {
    MsgBox num  ; Shows control object, not number!
}
Loop 3 {
    MyGui.Add("Button", , "Button " A_Index).OnEvent("Click", ShowNum.Bind(A_Index))
}
```

**Fix:** Use variadic parameter to ignore GUI params:

```ahk
; CORRECT
ShowNum(num, *) {  ; * captures and ignores extra params
    MsgBox num
}
Loop 3 {
    MyGui.Add("Button", , "Button " A_Index).OnEvent("Click", ShowNum.Bind(A_Index))
}
```

## Related Concepts

- docs/manuals/ahk2/explanation/closures.md - Deep dive into closure memory model
- docs/manuals/ahk2/reference/closure-behavior.md - Complete closure reference
- docs/source-reports/ahk2/ahk2_lang_06_closures.md - Original research on closures

## Key Takeaway

The loop closure problem stems from reference capture - closures store pointers to variables, not snapshots of values. The solution is always to create a new variable with a unique memory location for each iteration. Choose the pattern that best fits your use case: local variables for clarity, default parameters for compactness, or `.Bind()` for explicit binding.
