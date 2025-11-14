# Memory Management Architecture in AutoHotkey v2

## The Core Misconception

**AutoHotkey v2 does NOT have garbage collection.**

If you approach AHK2 expecting JavaScript-style garbage collection or Python's mark-and-sweep algorithm, you will create memory leaks. AHK2 uses **deterministic reference counting** without cycle detection—a fundamentally different memory model that requires explicit lifecycle management for complex object graphs.

This isn't a limitation to work around; it's an architectural design choice that provides predictable, immediate destruction semantics similar to C++ RAII (Resource Acquisition Is Initialization) or COM objects. Understanding this distinction is critical for writing leak-free code.

---

## How Reference Counting Actually Works

### The Basic Mechanism

Every object in AHK2 maintains an internal reference counter. This counter tracks how many active references point to the object. When the counter reaches zero, the object is **immediately and deterministically** destroyed.

#### Reference Operations

```
CREATE REFERENCE    →  counter += 1
RELEASE REFERENCE   →  counter -= 1
COUNTER == 0        →  destroy object, call __Delete()
```

Unlike garbage-collected languages where destruction timing is unpredictable (happens "eventually" during GC cycles), reference counting provides **immediate reclamation**. The moment the last reference disappears, the object is freed.

### Simple Reference Lifecycle

Let's trace a simple object's lifetime with a visual timeline:

```
TIME    CODE                    REFCOUNT    MEMORY STATE
────────────────────────────────────────────────────────────────
t0      obj := {}                  1        [Object @0x1234]

t1      obj2 := obj                2        [Object @0x1234]
                                             ↑         ↑
                                            obj       obj2

t2      obj := ""                  1        [Object @0x1234]
                                             ↑
                                            obj2

t3      obj2 := ""                 0        __Delete() called
                                            Memory freed
```

At `t3`, the reference counter hits zero and the object is **immediately destroyed**. There's no delay, no GC pause, no background cleanup thread. The destruction is synchronous with the `obj2 := ""` assignment.

### Reference Creation Events

Reference counts increment during these operations:

```ahk
; Variable assignment
obj := {}                    ; refcount: 1
another := obj               ; refcount: 2

; Array/Map insertion
arr := [obj]                 ; refcount: 3
myMap[key] := obj            ; refcount: 4

; Object property assignment
container.prop := obj        ; refcount: 5

; Function parameter passing (temporary)
ProcessObject(obj)           ; refcount: 6 during call
                             ; refcount: 5 after return

; Closure capture
fn := () => Process(obj)     ; refcount: 6 (captured!)
```

### Reference Release Events

Reference counts decrement during these operations:

```ahk
; Variable reassignment
another := ""                ; refcount decreases

; Variable goes out of scope
{
    temp := obj              ; refcount increases
}                            ; refcount decreases (scope exit)

; Array/Map removal
arr.RemoveAt(1)              ; refcount decreases
myMap.Delete(key)            ; refcount decreases

; Object property deletion
container.DeleteProp("prop") ; refcount decreases
```

### The __Delete Lifecycle Hook

When an object's reference count reaches zero, AHK2 calls its `__Delete()` meta-function (if defined) **before** freeing memory:

```ahk
class TrackedObject {
    __New(name) {
        this.name := name
        MsgBox("Created: " name)
    }

    __Delete() {
        MsgBox("Destroyed: " this.name)
    }
}

{
    obj := TrackedObject("MyObject")
    ; "Created: MyObject" displayed
}
; "Destroyed: MyObject" displayed immediately on scope exit
```

This deterministic destruction is powerful for resource management—you can reliably release file handles, close connections, or clean up system resources in `__Delete()`.

### Acyclic Object Graph Success Story

Reference counting works perfectly for tree-like structures without circular references:

```
ROOT OBJECT (refcount: 1)
    ├─→ CHILD A (refcount: 1)
    │       ├─→ GRANDCHILD A1 (refcount: 1)
    │       └─→ GRANDCHILD A2 (refcount: 1)
    └─→ CHILD B (refcount: 1)
            └─→ GRANDCHILD B1 (refcount: 1)

When ROOT is released:
    1. ROOT refcount: 1 → 0, __Delete() runs
    2. ROOT releases CHILD A and B
    3. CHILD A refcount: 1 → 0, __Delete() runs
    4. CHILD A releases GRANDCHILD A1 and A2
    5. ... cascade continues
    6. Entire tree freed in deterministic order
```

This cascading destruction happens **immediately and synchronously** when the root reference is released. No GC scanning, no mark-and-sweep phases.

---

## Why Circular References Leak Forever

### The Fundamental Problem

Reference counting has one critical weakness: **it cannot detect cycles**. When objects reference each other in a loop, their reference counts never reach zero, even when no external references exist.

AHK2 has **no cycle detector**. This isn't a missing feature to be added later—the developer explicitly stated it would require "far more work than I'm willing to put in any time soon." As of v2, nothing has changed. Circular references are **permanent memory leaks**.

### Circular Reference Memory Diagram

Let's visualize what happens with the simplest cycle:

```ahk
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child
child.parent := parent
```

**Memory state after circular reference creation:**

```
ACCESSIBLE REFERENCES:
    parent (variable) ──────┐
    child (variable) ───┐   │
                        ↓   ↓
              ┌─────────────────┐
              │  PARENT OBJECT  │  refcount: 2
              │  name: "Parent" │
              │  child: ────────┼───┐
              └─────────────────┘   │
                       ↑            │
                       │            │
                       │            ↓
              ┌─────────────────┐
              │  CHILD OBJECT   │  refcount: 2
              │  name: "Child"  │
              │  parent: ───────┼───┘
              └─────────────────┘
```

Both objects have refcount 2:
- PARENT: Referenced by `parent` variable + CHILD's `.parent` property
- CHILD: Referenced by `child` variable + PARENT's `.child` property

**Now release the external variables:**

```ahk
parent := ""
child := ""
```

**Memory state after releasing variables:**

```
ACCESSIBLE REFERENCES: (none!)

LEAKED "GARBAGE ISLAND":
              ┌─────────────────┐
              │  PARENT OBJECT  │  refcount: 1  ← LEAKED!
              │  name: "Parent" │
              │  child: ────────┼───┐
              └─────────────────┘   │
                       ↑            │
                       │            │
                       │            ↓
              ┌─────────────────┐
              │  CHILD OBJECT   │  refcount: 1  ← LEAKED!
              │  name: "Child"  │
              │  parent: ───────┼───┘
              └─────────────────┘
```

Both objects now have refcount 1:
- PARENT: Referenced only by CHILD's `.parent` property
- CHILD: Referenced only by PARENT's `.child` property

**This is a permanent memory leak.** No external reference exists, yet both objects remain allocated. They form an isolated "garbage island" that will persist until script termination.

### Why There's No Automatic Detection

Garbage-collected languages (JavaScript, Python, Java) use **mark-and-sweep** algorithms that periodically:
1. Start from "root" references (globals, stack variables)
2. Mark all reachable objects by traversing references
3. Sweep (delete) all unmarked objects

This approach detects cycles because unreachable cycles aren't marked in step 2.

**AHK2 doesn't do this.** There is no periodic scanning, no marking phase, no sweep phase. Reference counting is the **only** memory management mechanism, and it operates purely on local counter updates—it never performs global graph analysis.

Adding cycle detection would require:
- A global object registry
- Periodic graph traversal (performance cost)
- Complex heuristics to determine when to scan
- State tracking for all objects
- Significant runtime overhead

The AHK2 architecture deliberately avoids this complexity in favor of deterministic, zero-overhead reference counting.

### Multi-Object Cycles

Cycles aren't limited to two objects. Any loop in the reference graph creates a leak:

```
THREE-OBJECT CYCLE:

    A.next ──→ B.next ──→ C.next ──→ A

    All three objects have refcount ≥ 1
    All three leak when external references are removed
```

```
SELF-REFERENCE (simplest cycle):

    obj := {}
    obj.self := obj
    obj := ""           ; LEAKED! refcount still 1 from obj.self
```

Even a single object can leak if it references itself.

### The Official Warning

The AutoHotkey GUI documentation explicitly warns:

> "If EventObj itself contains a reference to the Gui, this would typically create a circular reference which **prevents** the Gui from being automatically destroyed."

Note the word **prevents**—not "delays," not "defers," but **prevents**. Circular references completely block destruction.

---

## How to Break Cycles Manually

Since AHK2 won't detect cycles, you must break them manually before releasing external references.

### Breaking the Parent-Child Cycle

**Before (leaks):**

```ahk
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child
child.parent := parent

parent := ""  ; LEAK: parent refcount still 1
child := ""   ; LEAK: child refcount still 1
```

**After (safe):**

```ahk
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child
child.parent := parent

; Break cycle FIRST
child.parent := ""      ; parent refcount: 2 → 1
parent := ""            ; parent refcount: 1 → 0, FREED
                        ; parent's __Delete releases child
child := ""             ; child refcount: 1 → 0, FREED
```

**Order matters!** You must break the cycle before releasing the variables holding external references.

### Automated Cleanup with __Delete

Implement `__Delete()` to break cycles automatically:

```ahk
class Node {
    __New(name) {
        this.name := name
        this.children := []
        this.parent := ""
    }

    AddChild(child) {
        this.children.Push(child)
        child.parent := this  ; Creates potential cycle
    }

    __Delete() {
        ; Break all circular references
        for child in this.children {
            if (child.parent = this)
                child.parent := ""
        }
        this.children := []
        this.parent := ""
    }
}

; Usage - now safe:
root := Node("Root")
child1 := Node("Child1")
child2 := Node("Child2")

root.AddChild(child1)
root.AddChild(child2)

root := ""  ; Triggers __Delete, breaks cycles, frees all nodes
```

When `root` is released, its `__Delete()` method runs, which:
1. Clears all `parent` references in children (breaks cycles)
2. Clears the `children` array (releases child references)
3. Allows children's refcounts to reach zero
4. Children's `__Delete()` methods run
5. Entire tree is freed

---

## How Closures Extend Lifetime

### Closure Capture Mechanics

Anonymous functions and bound methods create **closure objects** that capture variables from their defining scope. Critically, closures capture **by reference**, not by value.

When you write:

```ahk
largeData := Array(1000000)
fn := () => Process(largeData)
```

The closure `fn` doesn't copy `largeData`—it stores a **reference** to the `largeData` variable. This increments `largeData`'s reference count. As long as `fn` exists, `largeData` remains in memory.

### Closure Internal Structure

Conceptual representation:

```
CLOSURE OBJECT:
    ┌─────────────────────────┐
    │  Function Code Pointer  │
    ├─────────────────────────┤
    │  Captured Variables:    │
    │                         │
    │  • largeData ──────┐    │
    │                    │    │
    └────────────────────┼────┘
                         │
                         ↓
                  [Array object]  refcount: 2
                         ↑
                         │
                    largeData (variable)
```

The closure maintains an internal reference to `largeData`, incrementing its refcount. Setting `largeData := ""` decrements the count to 1, but the closure keeps it alive.

### Free Variable Sets

When a closure is created, AHK2 analyzes which outer-scope variables are referenced in the function body. These become the closure's **free variable set**:

```ahk
{
    data := [1, 2, 3]
    flag := true
    unrelated := "not used"

    ; Closure captures: data, flag (NOT unrelated)
    fn := () => (flag ? data : [])
}
; 'unrelated' goes out of scope and is freed
; 'data' and 'flag' remain alive (captured by fn)
```

Only variables actually used in the closure body are captured. But once captured, they're kept alive as long as the closure exists.

### The Timer Lifetime Extension Problem

Consider this common pattern:

```ahk
CreateWindow() {
    largeData := Array(1000000)
    gui := Gui()
    SetTimer(() => ProcessData(largeData), 1000)
    return gui
}

win := CreateWindow()
win := ""  ; GUI freed, but largeData LEAKED!
```

**Reference chain analysis:**

```
SCRIPT EVENT LOOP (global storage)
    │
    └─→ TIMER REGISTRY
            │
            └─→ TIMER CALLBACK (closure object)
                    │
                    └─→ largeData (captured)
                            │
                            └─→ [Array with 1,000,000 elements]
```

When `CreateWindow()` returns:
- `gui` is returned (refcount transferred to caller)
- `largeData` local variable goes out of scope
- **But** the closure `() => ProcessData(largeData)` captured `largeData`
- Timer stores the closure in global registry
- `largeData` refcount: 1 (held by closure)
- `largeData` **never freed** until timer is deleted

Setting `win := ""` releases the GUI but doesn't touch the timer. The timer is registered globally with the script's event loop, independent of any object.

### Timeline Visualization

```
TIME    EVENT                           largeData REFCOUNT
────────────────────────────────────────────────────────────
t0      largeData := Array(...)         1
t1      closure := () => Process(...)   2 (captured!)
t2      SetTimer(closure, 1000)         2 (timer stores closure)
t3      return from CreateWindow()      1 (local var released)
t4      win := ""                       1 (still held by closure)
t5      ... timer keeps running ...     1 (LEAKED until deleted)
```

### Proper Timer Cleanup

Store timer functions for explicit cleanup:

```ahk
CreateWindow() {
    largeData := Array(1000000)
    gui := Gui()

    timerFunc := () => ProcessData(largeData)
    SetTimer(timerFunc, 1000)

    ; Store reference for cleanup
    gui.timerFunc := timerFunc
    return gui
}

win := CreateWindow()

; Proper cleanup:
SetTimer(win.timerFunc, "Delete")  ; Remove from timer registry
win.timerFunc := ""                 ; Release closure
win := ""                           ; Now largeData can be freed
```

**Critical distinction:**
- `SetTimer(fn, "Off")` — Disables timer, but retains function reference (STILL LEAKS)
- `SetTimer(fn, "Delete")` — Removes timer, releases function reference (SAFE)
- `SetTimer(fn, 0)` — Also removes timer (equivalent to "Delete")

### For-Loop Closure Gotcha

Closures capture **variables**, not **values**:

```ahk
buttons := []
Loop 5 {
    i := A_Index
    btn := Gui.Add("Button", , "Button " i)
    btn.OnEvent("Click", () => MsgBox(i))
    buttons.Push(btn)
}
```

**What you might expect:**
- Button 1 shows "1"
- Button 2 shows "2"
- Button 3 shows "3"
- etc.

**What actually happens:**
- All buttons show "5"

**Why?** All five closures capture the **same variable** `i`. After the loop completes, `i` contains 5. When any button is clicked, the closure reads the current value of `i`, which is 5.

**Memory diagram after loop:**

```
CLOSURE 1 ─┐
CLOSURE 2 ─┼─→ i (variable) = 5
CLOSURE 3 ─┤
CLOSURE 4 ─┤
CLOSURE 5 ─┘
```

All closures share the same `i` reference.

**Solution: Capture value per iteration:**

```ahk
Loop 5 {
    i := A_Index
    btn := Gui.Add("Button", , "Button " i)

    ; Create intermediate function that captures 'value'
    captureValue := (value) => () => MsgBox(value)
    btn.OnEvent("Click", captureValue(i))

    buttons.Push(btn)
}
```

Or use a function parameter:

```ahk
Loop 5 {
    i := A_Index
    btn := Gui.Add("Button", , "Button " i)

    ; Pass i as parameter (creates new closure each iteration)
    btn.OnEvent("Click", ((index) => () => MsgBox(index))(i))

    buttons.Push(btn)
}
```

---

## Why Timers and Events Keep Objects Alive

### Global Registration Architecture

Timers, hotkeys, and event handlers aren't stored with objects—they're registered in **global script state**. This creates hidden reference chains that extend object lifetimes.

### Timer Storage in Script Runtime

Conceptual architecture:

```
SCRIPT RUNTIME (global)
    │
    ├─→ VARIABLE SCOPE STACK
    │       └─→ Local variables (managed per scope)
    │
    ├─→ TIMER REGISTRY (global Map)
    │       ├─→ Timer1: {callback: fn1, interval: 1000, enabled: true}
    │       ├─→ Timer2: {callback: fn2, interval: 500, enabled: false}
    │       └─→ Timer3: {callback: fn3, interval: 2000, enabled: true}
    │
    ├─→ HOTKEY REGISTRY (global Map)
    │       ├─→ "F1": {callback: hotkeyFn1, enabled: true}
    │       └─→ "^c": {callback: hotkeyFn2, enabled: true}
    │
    └─→ EVENT LOOP
            └─→ Polls timers, checks hotkeys, dispatches events
```

When you call `SetTimer(fn, 1000)`, the function object `fn` is stored in the global TIMER REGISTRY. It doesn't matter if `fn` was created in a local scope—the timer registry keeps it alive globally.

### Reference Chain Example

```ahk
class DataProcessor {
    __New() {
        this.data := Array(1000000)
        this.timerFunc := () => this.Process()
        SetTimer(this.timerFunc, 1000)
    }

    Process() {
        ; Work with this.data
    }
}

proc := DataProcessor()
proc := ""  ; LEAKED!
```

**Reference chain keeping DataProcessor alive:**

```
GLOBAL TIMER REGISTRY
    │
    └─→ timerFunc (closure)
            │
            └─→ 'this' (captured variable)
                    │
                    └─→ DataProcessor instance
                            │
                            ├─→ data (Array with 1,000,000 elements)
                            └─→ timerFunc (circular back to closure)
```

This creates a **reference cycle**:
- Timer → closure → `this` → DataProcessor → timerFunc → closure

The cycle prevents destruction even after `proc := ""` releases the external reference.

### Event Handler Reference Chain

Similar pattern with GUI events:

```ahk
class MyWindow {
    __New() {
        this.gui := Gui()
        this.data := LargeObject()
        this.gui.OnEvent("Close", () => this.OnClose())
    }

    OnClose() {
        ; Handle close
    }
}

win := MyWindow()
win := ""  ; LEAKED!
```

**Reference cycle:**

```
win (variable) ──→ MyWindow instance
                        │
                        ├─→ gui (Gui object)
                        │       │
                        │       └─→ Event handler (closure capturing 'this')
                        │                   │
                        │                   └─→ MyWindow instance (CYCLE!)
                        │
                        └─→ data (LargeObject)
```

Setting `win := ""` doesn't break the cycle. The GUI's event handler closure keeps `this` alive, which keeps the GUI alive, which keeps the closure alive.

### Hotkey Reference Chains

```ahk
class Controller {
    __New() {
        this.state := {}
        Hotkey("F1", () => this.Toggle())
    }

    Toggle() {
        this.state.active := !this.state.active
    }
}

ctrl := Controller()
ctrl := ""  ; LEAKED - hotkey still holds reference!
```

The hotkey remains registered globally until explicitly removed with `Hotkey("F1", "Off")`.

### Proper Cleanup Pattern

Always store callback references and explicitly unregister:

```ahk
class ManagedController {
    __New() {
        this.state := {}
        this.hotkeyFunc := () => this.Toggle()
        this.timerFunc := () => this.Update()

        Hotkey("F1", this.hotkeyFunc)
        SetTimer(this.timerFunc, 1000)
    }

    Toggle() {
        this.state.active := !this.state.active
    }

    Update() {
        ; Update state
    }

    Cleanup() {
        if this.HasProp("hotkeyFunc") {
            Hotkey("F1", "Off")
            this.hotkeyFunc := ""
        }

        if this.HasProp("timerFunc") {
            SetTimer(this.timerFunc, "Delete")
            this.timerFunc := ""
        }
    }

    __Delete() {
        this.Cleanup()
    }
}

ctrl := ManagedController()
ctrl.Cleanup()  ; Explicit cleanup
ctrl := ""      ; Now safe
```

---

## The GUI Special Case: Why Gui.Show() Increments Refcount

### The Problem This Solves

Consider this naive code:

```ahk
ShowDialog() {
    gui := Gui()
    gui.Add("Text", , "Hello")
    gui.Add("Button", , "OK").OnEvent("Click", (*) => gui.Destroy())
    gui.Show()
}

ShowDialog()  ; GUI appears, then IMMEDIATELY DISAPPEARS!
```

**Why?** When `ShowDialog()` returns:
1. Local variable `gui` goes out of scope
2. GUI object refcount drops to 0
3. GUI is immediately destroyed
4. Window vanishes

This is surprising behavior for users expecting the GUI to remain visible.

### The Architectural Solution

When you call `Gui.Show()`, AHK2 **increments the GUI object's reference count**. This creates a "self-reference" that keeps the GUI alive even after all external variables are released.

**Conceptual representation:**

```
BEFORE gui.Show():
    gui (variable) ──→ Gui object (refcount: 1)

AFTER gui.Show():
    gui (variable) ──→ Gui object (refcount: 2)
                            ↑
                            │
    GLOBAL GUI REGISTRY ────┘  (internal reference)
```

The GUI object now has two references:
1. Your `gui` variable
2. Internal reference from global GUI registry

When `gui` goes out of scope, refcount drops to 1 (not 0), and the GUI remains visible.

### When the Self-Reference Is Released

The internal reference is released when:
- `gui.Destroy()` is called
- `gui.Hide()` is called
- The window is closed by user action

```ahk
ShowDialog() {
    gui := Gui()
    gui.Add("Text", , "Hello")
    gui.Add("Button", , "OK").OnEvent("Click", (*) => gui.Destroy())
    gui.Show()  ; refcount: 2
}  ; gui variable released, refcount: 1, GUI STAYS VISIBLE

; User clicks OK button:
;   → gui.Destroy() called
;   → Internal reference released
;   → refcount: 0
;   → GUI object freed
```

### Why This Can Create Leaks

If you attach closures capturing the GUI to events, you create a cycle:

```ahk
gui := Gui()
gui.Add("Button", , "Click").OnEvent("Click", () => Process(gui))
gui.Show()
gui := ""  ; Doesn't free GUI due to cycle!
```

**Reference cycle:**

```
GLOBAL GUI REGISTRY ──→ Gui object
                            │
                            ├─→ Button
                            │       │
                            │       └─→ Event handler (closure)
                            │                   │
                            │                   └─→ gui (captured)
                            │                           │
                            └───────────────────────────┘
                                    (CYCLE!)
```

The GUI's internal reference + the closure's capture of `gui` create a permanent cycle.

### Safe GUI Patterns

**Pattern 1: Extend Gui class**

```ahk
class MyDialog extends Gui {
    __New() {
        super.__New(, "My Dialog", this)  ; Pass 'this' as event sink
        this.Add("Button", , "Click").OnEvent("Click", "OnClick")
        this.Show()
    }

    OnClick(btn, info) {
        ; 'this' refers to the Gui object
        MsgBox("Button clicked in " this.Title)
    }
}

dialog := MyDialog()  ; Safe - no closure cycle
dialog := ""          ; Safe to release (GUI still visible due to Show())
```

**Pattern 2: Store closure reference for cleanup**

```ahk
gui := Gui()
btn := gui.Add("Button", , "Click")

clickHandler := (*) => MsgBox("Clicked")
btn.OnEvent("Click", clickHandler)

gui.Show()

; Cleanup:
btn.OnEvent("Click", "")  ; Remove event handler
gui.Destroy()             ; Release internal reference
gui := ""                 ; Now fully freed
```

---

## Debunked Misconceptions

### Myth 1: "AHK2 Has Garbage Collection"

**FALSE.** AHK2 uses **reference counting**, not garbage collection. These are fundamentally different:

| Reference Counting (AHK2) | Garbage Collection (JS/Python) |
|---------------------------|-------------------------------|
| Deterministic (immediate) | Non-deterministic (eventual) |
| No cycle detection        | Detects cycles automatically |
| Zero runtime overhead     | Periodic scanning overhead |
| Manual cycle management   | Automatic cycle cleanup |
| Predictable `__Delete()` timing | Unpredictable finalizer timing |

If you assume garbage collection exists, you'll create circular references expecting automatic cleanup. **They will leak forever.**

### Myth 2: "Circular References Are Automatically Detected"

**FALSE.** AHK2 has **no cycle detector**. This is explicit design choice, not missing feature.

```ahk
; This LEAKS permanently:
a := {}
b := {}
a.ref := b
b.ref := a
a := ""
b := ""
; Both objects remain in memory until script exit
```

Languages with garbage collection (JavaScript, Python, Java, C#) use mark-and-sweep algorithms that detect unreachable cycles. AHK2 deliberately avoids this complexity.

**You must break cycles manually** before releasing external references.

### Myth 3: "SetTimer 'Off' Releases Memory"

**PARTIALLY FALSE.** `SetTimer(fn, "Off")` disables the timer but **keeps the function reference**:

```ahk
largeData := Array(1000000)
timerFunc := () => Process(largeData)
SetTimer(timerFunc, 1000)

SetTimer(timerFunc, "Off")  ; Timer stops, but largeData STILL IN MEMORY!
```

The timer registry still holds `timerFunc`, which captures `largeData`. To actually release memory:

```ahk
SetTimer(timerFunc, "Delete")  ; Removes timer AND releases function
timerFunc := ""                 ; Now largeData can be freed
```

Or equivalently:

```ahk
SetTimer(timerFunc, 0)  ; 0 is same as "Delete"
timerFunc := ""
```

### Myth 4: "Closures Capture By Value"

**FALSE.** Closures capture variables **by reference**:

```ahk
counter := 0
fn := () => ++counter

fn()  ; Returns 1
fn()  ; Returns 2
fn()  ; Returns 3

MsgBox(counter)  ; Shows 3 - closure modified original variable!
```

The closure `fn` holds a reference to the `counter` variable, not a copy of its value. Modifying `counter` through the closure changes the original variable.

This is critical for understanding lifetime extension:

```ahk
{
    data := LargeArray()
    fn := () => Process(data)
}
; 'data' is NOT freed - closure keeps it alive
```

### Myth 5: "Going Out of Scope Always Frees Memory"

**FALSE.** Variables going out of scope only **release one reference**. If other references exist, the object persists:

```ahk
global globalArray := []

CreateLeak() {
    obj := {data: "large"}
    globalArray.Push(obj)
}  ; 'obj' goes out of scope, but object persists in globalArray!

CreateLeak()
; Object still exists in globalArray
```

Additionally, closures extend scope indefinitely:

```ahk
CreateClosure() {
    data := "captured"
    return () => data
}

fn := CreateClosure()
; 'data' local variable went out of scope in CreateClosure()
; BUT closure keeps it alive!

fn()  ; Returns "captured" - still accessible!
```

### Myth 6: "__Delete Is Called When Objects Go Out of Scope"

**MISLEADING.** `__Delete` is called when **reference count reaches zero**, not when a specific variable goes out of scope:

```ahk
class Tracker {
    __New(name) => (this.name := name)
    __Delete() => MsgBox("Deleted: " this.name)
}

{
    obj := Tracker("A")
    another := obj
}  ; obj goes out of scope, but __Delete NOT called!
   ; 'another' still holds reference

another := ""  ; NOW __Delete is called
```

If any reference remains (in arrays, closures, object properties, globals), `__Delete` won't run.

### Myth 7: "ObjPtr Keeps Objects Alive"

**FALSE.** `ObjPtr()` returns an **unmanaged pointer** that doesn't affect reference counts:

```ahk
obj := {}
ptr := ObjPtr(obj)
obj := ""  ; Object freed immediately!

; ptr is now a DANGLING POINTER - object no longer exists!
; Using ObjFromPtr(ptr) will access freed memory (undefined behavior)
```

To keep an object alive while holding a pointer, use `ObjAddRef()`:

```ahk
obj := {}
ptr := ObjPtr(obj)
ObjAddRef(ptr)  ; Increment refcount

obj := ""  ; Object NOT freed (refcount still > 0)

; Later:
ObjRelease(ptr)  ; Decrement refcount, now object can be freed
```

**Warning:** Manual reference counting is dangerous. Use only for advanced COM interop.

### Myth 8: "GUI Objects Are Automatically Kept Alive"

**PARTIALLY TRUE.** Only **shown** GUIs have internal references:

```ahk
; This GUI is immediately destroyed:
gui := Gui()
gui.Add("Text", , "Hello")
gui := ""  ; Destroyed (never shown)

; This GUI persists:
gui := Gui()
gui.Add("Text", , "Hello")
gui.Show()  ; Creates internal reference
gui := ""   ; GUI stays visible (refcount still > 0)
```

Hidden GUIs (created but not shown) are destroyed normally when references are released.

---

## ASCII Diagrams: Complete Reference

### Reference Count Lifecycle

```
OBJECT LIFECYCLE:

Creation        Assignment      Scope Exit      Destruction
────────        ──────────      ──────────      ───────────
  New              obj2 := obj      }              refcount
  obj := {}         ↓               ↓              reaches 0
    ↓               ↓               ↓                  ↓
[refcount: 1]   [refcount: 2]   [refcount: 1]      __Delete()
    │               │               │                  │
    │               │               │                  ↓
    └───────────────┴───────────────┴─────────→   Memory freed
```

### Circular Reference Memory Structure

```
SIMPLE CYCLE (2 objects):

    ┌────────────────┐
    │   OBJECT A     │  refcount: 1
    │   .ref ────────┼──┐
    └────────────────┘  │
            ↑           │
            │           │
            │           ↓
    ┌────────────────┐
    │   OBJECT B     │  refcount: 1
    │   .ref ────────┼──┘
    └────────────────┘

    No external references exist
    Both objects leak forever


COMPLEX CYCLE (3 objects):

    ┌────────────────┐
    │   OBJECT A     │  refcount: 1
    │   .next ───────┼──┐
    └────────────────┘  │
            ↑           │
            │           ↓
            │    ┌────────────────┐
            │    │   OBJECT B     │  refcount: 1
            │    │   .next ───────┼──┐
            │    └────────────────┘  │
            │                        │
            │                        ↓
            │    ┌────────────────┐
            │    │   OBJECT C     │  refcount: 1
            └────┼── .next        │
                 └────────────────┘

    Three-object cycle
    All three leak when external refs released
```

### Closure Capture Architecture

```
CLOSURE OBJECT INTERNALS:

    SOURCE CODE:
    ───────────
    {
        data := Array(1000)
        count := 0
        fn := () => Process(data, count)
    }


    MEMORY LAYOUT:
    ─────────────
                            ┌─────────────────────┐
                            │   CLOSURE OBJECT    │
                            ├─────────────────────┤
                            │ Code: <function>    │
                            ├─────────────────────┤
                            │ Captured Variables: │
                            │                     │
    ┌───────────────────────┼─→ data ─────┐       │
    │                       │             │       │
    │   ┌───────────────────┼─→ count ──┐ │       │
    │   │                   └────────────┼─┼───────┘
    │   │                                │ │
    ↓   ↓                                ↓ ↓
[Array obj]                          [Number: 0]
refcount: 2                          refcount: 2
    ↑                                    ↑
    │                                    │
    data (local var)              count (local var)


When local scope exits:
    - data variable released (refcount: 2→1)
    - count variable released (refcount: 2→1)
    - Closure KEEPS both alive
    - Objects persist as long as closure exists
```

### Timer → Closure → Object Chain

```
TIMER KEEPING OBJECT ALIVE:

GLOBAL SCRIPT STATE:
    │
    ├─→ TIMER REGISTRY
    │       │
    │       └─→ Timer ID: 12345
    │               ├─── interval: 1000ms
    │               ├─── enabled: true
    │               └─── callback: ────────┐
    │                                      │
    └─→ VARIABLE SCOPES                    │
            (empty - locals out of scope)  │
                                           │
                                           ↓
                        ┌──────────────────────────┐
                        │   CLOSURE OBJECT         │
                        │                          │
                        │   Captured: 'myObj' ─────┼─┐
                        └──────────────────────────┘ │
                                                     │
                                                     ↓
                                        ┌────────────────────┐
                                        │  APPLICATION OBJ   │
                                        │                    │
                                        │  .data ────────────┼─┐
                                        │  .state ───────────┼─┤
                                        └────────────────────┘ │
                                                ↓              ↓
                                         [Large data]    [State obj]


REFERENCE CHAIN:
    Timer Registry → Closure → myObj → data/state

LEAK CONDITION:
    Even though 'myObj' local variable went out of scope,
    the timer keeps the entire chain alive indefinitely.

SOLUTION:
    SetTimer(closureFunc, "Delete")  ; Breaks chain at step 1
```

### GUI Event Handler Cycle

```
GUI CIRCULAR REFERENCE:

USER CODE:
    class MyWindow {
        __New() {
            this.gui := Gui()
            this.data := LargeData()
            this.gui.OnEvent("Close", () => this.OnClose())
        }
    }


MEMORY STRUCTURE:

    win (variable)
        │
        ↓
    ┌───────────────────────────────┐
    │    MyWindow instance          │  refcount: 2
    │                               │
    │    .gui ──────────────────────┼──┐
    │    .data ─────────────────┐   │  │
    └───────────────────────────┼───┘  │
                ↑               │      │
                │               │      │
                │               ↓      ↓
                │         [LargeData]  │
                │         refcount: 1  │
                │                      │
                │           ┌──────────┼──────────┐
                │           │    Gui object       │
                │           │                     │
                │           │  Event handlers:    │
                │           │    "Close": ────────┼──┐
                │           └─────────────────────┘  │
                │                                    │
                │                                    ↓
                │           ┌─────────────────────────────┐
                │           │   CLOSURE (OnEvent handler) │
                │           │                             │
                │           │   Captured: 'this' ─────────┼──┘
                └───────────┤                             │
                            └─────────────────────────────┘


REFERENCE CYCLE:
    MyWindow → Gui → Closure → MyWindow (CYCLE!)

RESULT:
    win := ""  releases external reference
    But refcount remains at 1 (cycle keeps it alive)
    PERMANENT LEAK


SOLUTION 1 - Break cycle manually:
    win.gui.OnEvent("Close", "")  ; Remove handler
    win.gui.Destroy()
    win := ""

SOLUTION 2 - Extend Gui instead:
    class MyWindow extends Gui {
        __New() {
            super.__New(, "Title", this)  ; No closure needed
        }
        OnClose(gui) { }  ; Method called directly
    }
```

---

## Advanced Patterns

### Weak Reference Pattern

For parent-child relationships without cycles, use unmanaged pointers:

```ahk
class WeakChild {
    __New(parent) {
        ; Store pointer, not reference (doesn't increment refcount)
        this.parentPtr := ObjPtr(parent)
    }

    GetParent() {
        try {
            ; ObjFromPtrAddRef attempts to create managed reference
            ; Will throw if parent was freed
            parent := ObjFromPtrAddRef(this.parentPtr)
            return parent
        } catch {
            return ""  ; Parent no longer exists
        }
    }

    UseParent() {
        parent := this.GetParent()
        if (parent) {
            ; Safe to use parent
            MsgBox(parent.name)
        } else {
            MsgBox("Parent was freed")
        }
    }
}

class Parent {
    __New(name) {
        this.name := name
        this.children := []
    }

    AddChild(child) {
        this.children.Push(child)
        ; Child doesn't hold reference to parent (weak pointer)
        ; No circular reference created!
    }
}

parent := Parent("Root")
child := WeakChild(parent)
parent.AddChild(child)

parent := ""  ; Parent freed immediately (no cycle)
child.UseParent()  ; Shows "Parent was freed"
```

**Warning:** This is advanced technique requiring careful lifetime management. Parent can be freed while children still exist.

### Subscription Manager Pattern

For event-driven architectures, use subscription manager to auto-cleanup:

```ahk
class Subscriber {
    __New(eventBus) {
        this.eventBus := eventBus
        this.subscriptions := []
    }

    Subscribe(eventName, callback) {
        id := this.eventBus.On(eventName, callback)
        this.subscriptions.Push({event: eventName, id: id})
        return id
    }

    UnsubscribeAll() {
        for sub in this.subscriptions {
            this.eventBus.Off(sub.event, sub.id)
        }
        this.subscriptions := []
    }

    __Delete() {
        this.UnsubscribeAll()
    }
}

class Component {
    __New(bus) {
        this.subscriber := Subscriber(bus)
        this.data := []

        ; Subscriptions auto-cleaned on destruction
        this.subscriber.Subscribe("update", (d) => this.OnUpdate(d))
        this.subscriber.Subscribe("clear", (*) => this.OnClear())
    }

    OnUpdate(data) => this.data.Push(data)
    OnClear() => (this.data := [])
}

; Usage - no manual cleanup needed:
bus := EventBus()
comp := Component(bus)
comp := ""  ; Subscriber.__Delete() auto-unsubscribes
```

### Resource Handle Pattern

For system resources (files, handles, buffers):

```ahk
class FileHandle {
    __New(path, mode := "r") {
        this.handle := FileOpen(path, mode)
        if (!this.handle)
            throw Error("Failed to open: " path)
    }

    Read(bytes := -1) {
        if (!this.handle)
            throw Error("File already closed")
        return this.handle.Read(bytes)
    }

    Write(text) {
        if (!this.handle)
            throw Error("File already closed")
        return this.handle.Write(text)
    }

    Close() {
        if (this.handle) {
            this.handle.Close()
            this.handle := ""
        }
    }

    __Delete() {
        this.Close()
    }
}

; Deterministic resource cleanup:
{
    file := FileHandle("data.txt", "r")
    content := file.Read()
}  ; file.__Delete() called immediately, handle closed
```

---

## Memory Leak Detection Strategy

### Manual Reference Counting (Debug Only)

```ahk
GetRefCount(&obj) {
    ; WARNING: For debugging only - never use in production
    ptr := ObjPtr(obj)
    ObjAddRef(ptr)
    return ObjRelease(ptr)
}

myObj := {}
MsgBox("Initial: " GetRefCount(&myObj))  ; Shows 1

another := myObj
MsgBox("After assignment: " GetRefCount(&myObj))  ; Shows 2

another := ""
MsgBox("After release: " GetRefCount(&myObj))  ; Shows 1
```

**Never use in production code.** Misuse causes crashes.

### Process Memory Monitoring

```ahk
#Requires AutoHotkey v2.0

GetProcessMemoryKB(PID) {
    size := 440
    pmcex := Buffer(size, 0)
    hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", PID, "Ptr")

    if (!hProcess)
        return 0

    result := 0
    if (DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size))
        result := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt")

    DllCall("CloseHandle", "Ptr", hProcess)
    return result
}

; Leak detection test
myPID := DllCall("GetCurrentProcessId")
startMem := GetProcessMemoryKB(myPID)

OutputDebug("Starting memory: " startMem " KB")

Loop 10000 {
    ; Function you're testing for leaks
    TestFunction()

    ; Sample every 1000 iterations
    if (Mod(A_Index, 1000) = 0) {
        currentMem := GetProcessMemoryKB(myPID)
        delta := currentMem - startMem
        OutputDebug("Iteration " A_Index ": " currentMem " KB (+" delta " KB)")
    }
}

endMem := GetProcessMemoryKB(myPID)
totalLeak := endMem - startMem
MsgBox("Memory increase: " totalLeak " KB")

; If totalLeak grows linearly with iterations → LEAK
; If totalLeak plateaus after initial allocations → probably safe
```

### Leak Test Pattern

```ahk
TestForLeaks(testFunc, iterations := 10000, sampleRate := 1000) {
    myPID := DllCall("GetCurrentProcessId")
    startMem := GetProcessMemoryKB(myPID)

    samples := []

    Loop iterations {
        testFunc()

        if (Mod(A_Index, sampleRate) = 0) {
            currentMem := GetProcessMemoryKB(myPID)
            delta := currentMem - startMem
            samples.Push({iteration: A_Index, memory: currentMem, delta: delta})

            OutputDebug("Iteration " A_Index ": +" delta " KB")
        }
    }

    ; Analyze trend
    if (samples.Length >= 3) {
        firstDelta := samples[1].delta
        lastDelta := samples[-1].delta
        growth := lastDelta - firstDelta

        if (growth > 1000) {  ; More than 1MB growth
            MsgBox("LEAK DETECTED!`nMemory growth: " growth " KB`nLikely leaking memory")
        } else {
            MsgBox("Probably safe.`nMemory growth: " growth " KB`nLikely initial allocations")
        }
    }
}

; Usage:
TestForLeaks(() => CreateCircularRef())

CreateCircularRef() {
    a := {}
    b := {}
    a.ref := b
    b.ref := a
    ; a and b leak here
}
```

---

## Summary: Key Architectural Principles

### 1. Reference Counting, Not Garbage Collection

AHK2 uses **deterministic reference counting** without cycle detection. Objects are freed immediately when refcounts reach zero. There is no mark-and-sweep, no generational collection, no background GC thread.

### 2. Circular References Leak Forever

Any cycle in the object graph creates a permanent memory leak. You must manually break cycles before releasing external references. No automatic detection exists.

### 3. Closures Capture By Reference

Anonymous functions and bound methods capture variables by reference, not by value. Captured variables remain alive as long as the closure exists.

### 4. Global Registration Extends Lifetime

Timers, hotkeys, and event handlers are stored in global registries. Objects referenced by these callbacks remain alive until explicitly unregistered.

### 5. Deterministic Destruction Enables RAII

Predictable `__Delete()` timing allows resource cleanup patterns similar to C++ destructors. Use this for file handles, connections, and cleanup logic.

### 6. Explicit Lifecycle Management Required

For complex objects with timers, events, or circular references, implement explicit cleanup methods. Don't rely on automatic behavior—manually break reference chains.

### 7. Gui.Show() Creates Internal Reference

Shown GUIs have hidden self-references preventing premature destruction. This helps keep windows visible but can create cycles if combined with closure-based event handlers.

---

## Further Reading

For practical implementation guidance, see:
- `docs/manuals/ahk2/how-to/prevent-memory-leaks.md` - Step-by-step leak prevention
- `docs/manuals/ahk2/reference/object-model.md` - Object system technical details
- `docs/manuals/ahk2/reference/built-in-functions.md` - ObjAddRef, ObjRelease, ObjPtr reference

For foundational concepts:
- `docs/manuals/ahk2/explanation/closure-mechanics.md` - Deep dive into closure internals
- `docs/manuals/ahk2/explanation/event-system-architecture.md` - Event-driven patterns
