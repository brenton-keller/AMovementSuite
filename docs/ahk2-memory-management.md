# AHK2 Memory Management for A Movement Suite

**Critical knowledge for preventing memory leaks in this long-running background application.**

---

## The #1 Thing to Know

**AutoHotkey v2 does NOT have garbage collection.**

It uses **reference counting**, which means:
- Objects are freed **immediately** when reference count reaches 0
- **Circular references leak forever** (no cycle detector exists)
- You must manually break cycles in `__Delete()`
- Closures capture by **reference**, not by value

### Why This Matters for This Project

A Movement Suite runs indefinitely in the background and creates:
- **GUI overlays** (preview windows, grid positioning, selection highlights)
- **Timers** (continuous key checking, tooltips, cleanup tasks)
- **Event handlers** (GUI events, window messages)
- **Global state** (window grouping, feature toggles)

Without proper memory management, **memory will leak** over hours/days of continuous operation.

---

## SetTimer: The Critical Distinction

### ⚠️ CRITICAL: "Off" vs "Delete" vs 0

The most common source of memory leaks in AHK2 is misunderstanding `SetTimer`:

| Command | Effect | Memory State | Use When |
|---------|--------|--------------|----------|
| `SetTimer(func, "Off")` | Disable timer | **Function STILL HELD in memory!** | Temporary pause (will restart) |
| `SetTimer(func, "Delete")` | Remove timer completely | Function reference released | Permanent cleanup |
| `SetTimer(func, 0)` | Remove timer completely | Function reference released | Permanent cleanup (same as "Delete") |

### The Memory Leak

```ahk
; WRONG - Memory leak!
largeData := Array(1000000)
timerFunc := () => Process(largeData)
SetTimer(timerFunc, 1000)

; Later, trying to clean up:
SetTimer(timerFunc, "Off")  ; ❌ Timer stops, but largeData STILL IN MEMORY!
timerFunc := ""             ; ❌ Timer system still holds the function reference
```

**Result:** `largeData` remains in memory until script exit.

### The Correct Pattern

```ahk
; CORRECT - Proper cleanup
largeData := Array(1000000)
timerFunc := () => Process(largeData)
SetTimer(timerFunc, 1000)

; Cleanup:
SetTimer(timerFunc, "Delete")  ; ✓ Remove timer, release function
timerFunc := ""                 ; ✓ Now largeData can be freed
```

Or equivalently:
```ahk
SetTimer(timerFunc, 0)  ; 0 is same as "Delete"
timerFunc := ""
```

### Examples from This Codebase

#### ✅ EXCELLENT: Using 0 to delete
```ahk
// windows_explorer.ahk2:63
SetTimer CheckForExplorer, 0
```

This properly removes the timer and frees memory.

#### ✅ GOOD: Single-execution timers
```ahk
// WindowMove.ahk2:22
SetTimer () => ToolTip("", , , 1), -1000
```

Negative period = single execution, then auto-deleted. No cleanup needed!

#### ✅ GOOD: Bounded lifetime with cleanup
```ahk
// MoveUtils.ahk2:733
SetTimer(DestroyHighlights.Bind(highlightGuis), -5000)
```

Single execution after 5 seconds, captures `highlightGuis` only temporarily.

#### ⚠️ NEEDS REVIEW: Continuous timers
```ahk
// WindowScaleXY.ahk2:79
SetTimer CheckKeysAndResize, 16  // Runs every 16ms forever
```

**Question:** How is this cleaned up when XY scaling is disabled?

**Current status:** Uses named function (not closure), so no direct leak. But timer continues running even when `xyScalingEnabled := false`.

**Potential improvement:** Stop timer when feature is disabled:
```ahk
ToggleXYScaling(*) {
    global xyScalingEnabled
    xyScalingEnabled := !xyScalingEnabled

    if (!xyScalingEnabled) {
        ; Consider: SetTimer CheckKeysAndResize, 0
        ; But need to restart when re-enabled
    }
}
```

---

## GUI Object Lifecycle

### The GUI Self-Reference Behavior

When you call `Gui.Show()`, AHK2 **increments the GUI's reference count** internally:

```
BEFORE gui.Show():
    gui (variable) → Gui object (refcount: 1)

AFTER gui.Show():
    gui (variable) → Gui object (refcount: 2)
                         ↑
    GLOBAL GUI REGISTRY ─┘  (internal reference)
```

This keeps the GUI alive even after your variable goes out of scope.

### When the Self-Reference is Released

The internal reference is removed when:
- `gui.Destroy()` is called
- `gui.Hide()` is called
- User closes the window

### How This Applies to This Project

Our temporary GUIs (preview overlays, grid positioning, selection highlights) are shown briefly:

```ahk
// Conceptual example from WindowScaleXY
preview := Gui("+AlwaysOnTop -Caption +ToolWindow")
preview.Show()  // Adds self-reference (refcount: 2)

// Later...
preview.Destroy()  // Removes self-reference, allows cleanup
```

**CRITICAL:** Always call `.Destroy()` on temporary GUIs when done!

### Cleanup Pattern

```ahk
// ✅ CORRECT
CreatePreviewGUI() {
    preview := Gui("+AlwaysOnTop")
    preview.Show()
    return preview
}

previewGui := CreatePreviewGUI()
// ... use preview ...
previewGui.Destroy()  // ← CRITICAL!
previewGui := ""
```

### Where This Matters in Our Codebase

- **Grid positioning GUI** (`WindowMove.ahk2`) - Shown on Z key press
- **Preview overlays** (`WindowScaleXY.ahk2`) - Shown during proportional scaling
- **Highlight overlays** (`MoveUtils.ahk2`, `WindowGrouping.ahk2`) - Selection highlights

**Review needed:** Verify all temporary GUIs call `.Destroy()` in cleanup paths.

---

## Closure Capture: How Lifetimes Extend

### The Rule: Closures Capture by Reference

When you create a closure (anonymous function), it captures variables **by reference**, not by value:

```ahk
counter := 0
fn := () => ++counter  // Captures reference to 'counter' variable

fn()  // Returns 1
fn()  // Returns 2
MsgBox(counter)  // Shows 2 - closure modified original!
```

### Lifetime Extension

```ahk
{
    largeData := Array(1000000)
    fn := () => Process(largeData)
    SetTimer(fn, 1000)
}
// 'largeData' local variable went out of scope
// BUT timer's closure keeps it alive indefinitely!
```

The timer keeps the closure alive, the closure keeps `largeData` alive. Memory leak!

### Examples from This Codebase

#### ✅ SAFE: Named functions (no capture)
```ahk
// WindowScaleXY.ahk2:79
SetTimer CheckKeysAndResize, 16
```

`CheckKeysAndResize` is a named function, not a closure. It accesses globals directly, doesn't capture anything. Safe!

#### ✅ SAFE: Single-execution closures
```ahk
// WindowMove.ahk2:22
SetTimer () => ToolTip("", , , 1), -1000
```

Closure executes once (negative period), then is auto-deleted. Captured variables freed after 1 second.

#### ✅ SAFE: Bounded capture
```ahk
// MoveUtils.ahk2:733
SetTimer(DestroyHighlights.Bind(highlightGuis), -5000)
```

Captures `highlightGuis` for 5 seconds only (single execution).

#### ⚠️ WARNING: Loop closures share variables
```ahk
// WRONG - All closures share same 'i'
Loop 5 {
    i := A_Index
    btn := gui.Add("Button", , "Button " i)
    btn.OnEvent("Click", () => MsgBox(i))  // All show "5"!
}

// RIGHT - Capture value per iteration
Loop 5 {
    btn := gui.Add("Button", , "Button " A_Index)
    btn.OnEvent("Click", (ctrl, *) => MsgBox(ctrl.Text))  // Use parameter
}
```

---

## Circular References Leak Forever

### The Problem

AHK2 has **no garbage collector** and **no cycle detector**. Any circular reference is a permanent memory leak:

```ahk
// LEAKS FOREVER
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child  // parent → child
child.parent := parent // child → parent (CYCLE!)

parent := ""  // parent refcount: 1 (child.parent still holds it)
child := ""   // child refcount: 1 (parent.child still holds it)

// Both objects leak until script exit
```

### Common Cycle Patterns

#### 1. GUI Event Handler Closures
```ahk
// LEAKS
gui := Gui()
gui.OnEvent("Close", () => HandleClose(gui))  // Captures 'gui'
gui.Show()
gui := ""  // LEAK: gui → event → closure → gui (cycle!)
```

**Solution:** Don't capture the GUI in the handler:
```ahk
gui := Gui()
gui.OnEvent("Close", (*) => ExitApp())  // No capture
gui.Show()
```

#### 2. Parent-Child Object References
```ahk
// LEAKS
class Node {
    __New(parent := "") {
        this.parent := parent
        this.children := []
        if (parent)
            parent.children.Push(this)  // Cycle: parent ↔ children
    }
}

// FIX: Break cycle in __Delete
class Node {
    __New(parent := "") {
        this.parent := parent
        this.children := []
        if (parent)
            parent.children.Push(this)
    }

    __Delete() {
        ; Break cycles before destruction
        for child in this.children
            child.parent := ""
        this.children := []
        this.parent := ""
    }
}
```

### Where This Could Affect This Project

**Window Grouping Feature** (`WindowGrouping.ahk2`):
- Stores references to two grouped windows
- Windows could theoretically have references back to the grouping system
- **Current status:** Likely safe (windows are just HWNDs, not objects)
- **Risk:** If we add window-specific state objects, could create cycles

**GUI Event Handlers:**
- Grid positioning GUI
- Preview overlays
- Settings dialogs

**Review:** Ensure no closures capture the GUI object they're attached to.

---

## Memory Leak Prevention Checklist

Use this when adding new features or reviewing existing code:

### Timers
- [ ] All continuous timers use named functions (not closures capturing large data) OR
- [ ] Timer function reference stored for later cleanup
- [ ] Cleanup uses `SetTimer(fn, 0)` or `SetTimer(fn, "Delete")` (never "Off")
- [ ] Single-use timers use negative period (e.g., `-1000`) for auto-cleanup
- [ ] Disabled features stop their timers (don't just skip logic)

### GUIs
- [ ] All temporary GUIs call `.Destroy()` when done
- [ ] `.Destroy()` is called in all exit paths (normal close, error, cancel)
- [ ] Long-lived GUIs don't capture themselves in event handlers
- [ ] Event handlers use named methods or non-capturing closures

### Closures
- [ ] Closures don't capture large objects unless necessary
- [ ] Closure lifetime is bounded (single-use timers, event cleanup)
- [ ] Loop closures capture values, not shared variables
- [ ] No circular references (GUI → closure → GUI)

### Global State
- [ ] Global objects cleaned up when features disabled
- [ ] Window references validated before use (window may close)
- [ ] Maps/Arrays cleaned up, not just set to empty

### Testing
- [ ] Run feature 1000 times, verify memory doesn't grow linearly
- [ ] Monitor process memory in Task Manager during extended use
- [ ] Test feature enable/disable cycles for leaks

---

## Debugging Memory Issues

### Quick Memory Check

```ahk
; Add to script for debugging
#F12::{
    pid := DllCall("GetCurrentProcessId")
    size := 440
    pmcex := Buffer(size, 0)
    hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", pid, "Ptr")

    if (hProcess) {
        DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size)
        memKB := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt")
        DllCall("CloseHandle", "Ptr", hProcess)

        MsgBox("Memory: " . Round(memKB/1024, 2) . " MB")
    }
}
```

Press Win+F12 to check current memory usage.

### Leak Testing Pattern

```ahk
; Test if a feature leaks memory
TestFeatureLeak() {
    pid := DllCall("GetCurrentProcessId")
    startMem := GetMemoryKB(pid)

    ; Repeat operation many times
    Loop 1000 {
        TriggerFeature()  ; Your feature here

        if (Mod(A_Index, 100) = 0) {
            currentMem := GetMemoryKB(pid)
            delta := currentMem - startMem
            OutputDebug("Iteration " . A_Index . ": +" . delta . " KB")
        }
    }

    endMem := GetMemoryKB(pid)
    MsgBox("Total memory growth: " . (endMem - startMem) . " KB")
}
```

If memory grows linearly with iterations → leak!
If memory plateaus after initial allocations → probably safe.

---

## Common Misconceptions

### ❌ "AHK2 has garbage collection"
**FALSE.** AHK2 uses reference counting. No mark-and-sweep, no cycle detection.

### ❌ "SetTimer 'Off' releases memory"
**FALSE.** Only `"Delete"` or `0` actually removes the timer and releases the function reference.

### ❌ "Closures capture by value"
**FALSE.** Closures capture variables **by reference**. Modifying captured variables affects the original.

### ❌ "Going out of scope always frees memory"
**FALSE.** If other references exist (in closures, timers, global arrays), the object persists.

### ❌ "GUI objects are automatically cleaned up"
**PARTIALLY TRUE.** Only **shown** GUIs have internal references. Hidden GUIs are freed normally.

---

## Further Reading

For deeper understanding, see the comprehensive AHK2 reference guide:

- `docs/reference/ahk2/reference/memory-management.md` - Complete reference
- `docs/reference/ahk2/explanation/memory-management-architecture.md` - Deep dive into the "why"
- `docs/reference/ahk2/how-to/prevent-memory-leaks.md` - Step-by-step prevention
- `docs/reference/ahk2/how-to/debug-memory-leaks.md` - Debugging techniques

---

## Summary: Key Takeaways

1. **AHK2 uses reference counting, not garbage collection**
2. **Circular references leak forever - break them manually**
3. **Use `SetTimer(fn, 0)` or `SetTimer(fn, "Delete")` for cleanup (never "Off")**
4. **Always `.Destroy()` temporary GUIs**
5. **Closures capture by reference and extend variable lifetime**
6. **Test long-running features for memory growth**

When in doubt: **explicitly clean up resources** rather than assuming automatic behavior.
